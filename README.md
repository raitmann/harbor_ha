# harbor_ha
Ansible role to deploy HA Harbor in air gapped environment


```
# Функция для нормализации имени образа
normalize_image_name() {
    echo "$1" | sed -E '
        s|^https?://||;   # Удаляем http:// или https://
        s|//+|/|g;        # Заменяем множественные слеши на один
        s|^/||;           # Удаляем ведущий слеш
        s|/$||            # Удаляем завершающий слеш
    '
}

# Функция для очистки тегов
clean_tag() {
    local tag=$1
    # Заменяем недопустимые символы на дефисы, удаляем двойные дефисы
    echo "$tag" | sed -E '
        s/[^a-zA-Z0-9._-]/-/g;
        s/--+/-/g;
        s/^-//;
        s/-$//
    ' | cut -c -128  # Обрезаем до 128 символов (ограничение Docker)
}

# Функция для создания проекта в Harbor
create_harbor_project() {
    local project=$1
    echo "Создание проекта ${project} в Harbor..."
    
    response=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
        -H "Content-Type: application/json" \
        -u "${HARBOR_USER}:${HARBOR_PASS}" \
        "${HARBOR_URL}/api/v2.0/projects" \
        -d '{"project_name": "'"${project}"'", "public": false}')
    
    [ "$response" -eq 201 ] && echo "Проект создан" || echo "Проект уже существует (код $response)"
}

# Функция для копирования образа
copy_image() {
    local repo=$1
    local image_path=$2
    local original_tag=$3
    
    # Нормализуем имена
    local clean_repo=$(normalize_image_name "$repo")
    local clean_image=$(normalize_image_name "$image_path")
    local clean_tag=$(clean_tag "$original_tag")
    
    # Формируем полные имена образов
    local source_image="${NEXUS_URL}/repository/${clean_repo}/${clean_image}:${original_tag}"
    local target_image="${HARBOR_URL}/${clean_repo}/${clean_image}:${clean_tag}"
    
    echo "Копирование: ${source_image} -> ${target_image}"
    
    # Pull из Nexus
    if ! echo "${NEXUS_PASS}" | docker login "${NEXUS_URL}" -u "${NEXUS_USER}" --password-stdin; then
        echo "[ERROR] Ошибка аутентификации в Nexus"
        return 1
    fi
    
    if ! docker pull "${source_image}"; then
        echo "[ERROR] Не удалось загрузить ${source_image}"
        docker logout "${NEXUS_URL}"
        return 1
    fi
    
    # Tag для Harbor
    docker tag "${source_image}" "${target_image}"
    
    # Push в Harbor
    if ! echo "${HARBOR_PASS}" | docker login "${HARBOR_URL}" -u "${HARBOR_USER}" --password-stdin; then
        echo "[ERROR] Ошибка аутентификации в Harbor"
        docker logout "${NEXUS_URL}"
        return 1
    fi
    
    if docker push "${target_image}"; then
        echo "[SUCCESS] Успешно скопирован ${target_image}"
    else
        echo "[ERROR] Не удалось загрузить в Harbor ${target_image}"
    fi
    
    # Очистка
    docker rmi -f "${source_image}" "${target_image}" 2>/dev/null
    docker logout "${NEXUS_URL}"
    docker logout "${HARBOR_URL}"
}

# Основной процесс
echo "=== Начало миграции $(date) ==="

# Получаем список Docker репозиториев
DOCKER_REPOS=$(curl -s -u "${NEXUS_USER}:${NEXUS_PASS}" \
    "https://${NEXUS_URL}/service/rest/v1/repositories" | \
    jq -r '.[] | select(.format == "docker") | .name')

for repo in $DOCKER_REPOS; do
    echo "Обработка репозитория: ${repo}"
    
    # Создаем проект в Harbor
    if [ "$CREATE_HARBOR_PROJECTS" = true ]; then
        create_harbor_project "${repo}"
    fi
    
    # Получаем образы из репозитория
    IMAGES=$(curl -s -u "${NEXUS_USER}:${NEXUS_PASS}" \
        "https://${NEXUS_URL}/service/rest/v1/components?repository=${repo}" | \
        jq -r '.items[] | select(.format == "docker") | .name + ":" + .version')
    
    for image in $IMAGES; do
        IFS=':' read -r image_path tag <<< "$image"
        image_name=$(normalize_image_name "${image_path#repository/${repo}/}")
        
        copy_image "${repo}" "${image_name}" "${tag}"
    done
done

echo "=== Миграция завершена $(date) ==="

```
