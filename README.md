#!/bin/bash
set -e

function cleanup() {
  rm -rf "${INSTALL_DIR}/data/cloned-files"
  rm -rf "${INSTALL_DIR}/data/webauthor/backup/"
  rm -rf "${INSTALL_DIR}/data/backup_metadata"
}
trap cleanup EXIT

ADMIN_DIR=$(dirname "${BASH_SOURCE[0]}")
source "${ADMIN_DIR}/.admin-functions.sh"
INSTALL_DIR=$(get_install_dir)

fix_various
ensure_root "Creating a backup requires root privileges."

FUSION_STARTED_BY_SCRIPT="false"
if ! is_fusion_running; then
  start_fusion
  FUSION_STARTED_BY_SCRIPT="true"
fi

HAD_ERRORS=0

pushd "${INSTALL_DIR}" 1> /dev/null
  echo -n "Creating database backup... "
    BACKUP_OUTPUT=$(docker-compose exec -T database-backup node /scripts/pg_backup "$1" | grep "Backup complete")
    if [ -z "${BACKUP_OUTPUT}" ]; then
      echo "Failed to create database backup"
      exit 1
    fi

    BACKUP_NAME=$(printf '%s' "${BACKUP_OUTPUT}" | awk -F ': ' '{print $2}')
    if [ -z "${BACKUP_NAME}" ]; then
      echo "Failed to determine backup name"
      exit 1
    fi
  echo "Done"

  echo "Creating file storage backup... "
    mkdir -p "${INSTALL_DIR}/data/cloned-files"
    rm -rf "${INSTALL_DIR}/data/cloned-files"/*

    mkdir "${INSTALL_DIR}/data/cloned-files/tasks/"
    mkdir "${INSTALL_DIR}/data/cloned-files/projects_dir/"
    
    chown -R --reference="${INSTALL_DIR}/data/temp/tasks" "${INSTALL_DIR}/data/cloned-files"

    docker-compose run -T --rm --no-deps \
        -u oxygen \
        -v "${INSTALL_DIR}/data/temp/tasks":/tasks \
        -v "${INSTALL_DIR}/data/cloned-files/tasks":/cloned-tasks \
      documents-storage bash /cli/exec.sh clone-tasks --source=/tasks --destination=/cloned-tasks

    set +e
    docker-compose run -T --rm --no-deps \
        -u oxygen \
        -v "${INSTALL_DIR}/data/cloned-files/projects_dir":/cloned-projects_dir \
      documents-storage bash /cli/exec.sh download-projects --destination=/cloned-projects_dir
    if [ $? -ne 0 ]; then
      HAD_ERRORS=1
    fi
    set -e
  echo "Done"
  

  # Creating the Web Author backup
  echo -n "Creating Web Author backup... "
    WEB_AUTHOR_BACKUP_LOCATION="${INSTALL_DIR}/data/webauthor/backup"

    rm -rf ${WEB_AUTHOR_BACKUP_LOCATION}
    mkdir -p ${WEB_AUTHOR_BACKUP_LOCATION}

    chown -R --reference="${INSTALL_DIR}/data/webauthor" ${WEB_AUTHOR_BACKUP_LOCATION}/

    docker-compose exec webauthor bash /cli/create-backup.sh /backup

    # get the container id of the running webauthor service
    WEBAUTHOR_CONTAINER_ID=$(docker-compose ps -q webauthor)
    # copy from container to backup dir on host
    docker cp -a ${WEBAUTHOR_CONTAINER_ID}:/backup/. ${WEB_AUTHOR_BACKUP_LOCATION}

  echo "Done"
  
popd 1> /dev/null


if [ "${FUSION_STARTED_BY_SCRIPT}" == "true" ]; then
  stop_fusion
fi

# Trim leading & tailing white space
# shellcheck disable=SC2001
BACKUP_NAME=$(sed 's/^[[:space:]]*//' <<< "${BACKUP_NAME}")
# shellcheck disable=SC2001
BACKUP_NAME=$(sed 's/[[:space:]]*$//' <<< "${BACKUP_NAME}")

BACKUP_METADATA_FILE="${INSTALL_DIR}/data/backup_metadata"
cat "${INSTALL_DIR}/default-data/version" > "${BACKUP_METADATA_FILE}"
printf '%s\n' "BK_META_BACKUP_NAME=${BACKUP_NAME}" >> "${BACKUP_METADATA_FILE}"

echo -n "Creating Content Fusion backup... "
set +e
tar --warning=no-file-changed -zc --exclude="./temp" --transform="s|./cloned-files/|./|" -f ./fusion-backup.tar.gz -C "${INSTALL_DIR}/data" .
if [ $? -ne 0 ]; then
  echo "Backup completed with errors"
  HAD_ERRORS=1
fi
set -e
echo "Done"

echo "Backup created: '`pwd`/fusion-backup.tar.gz'"

if [ "${HAD_ERRORS}" -eq 1 ]; then
  echo "WARNING: There were errors during the backup process. Please check the messages above."
  exit 1
fi
