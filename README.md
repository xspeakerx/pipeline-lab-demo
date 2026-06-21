# Tekton / OpenShift Pipelines — шпаргалка для EX288

## Иерархия объектов

```
Task / ClusterTask  →  TaskRun
Pipeline             →  PipelineRun
EventListener + Trigger + TriggerBinding + TriggerTemplate  →  (автозапуск PipelineRun)
```

---

## 1. Task — атомарная единица работы

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: my-task
spec:
  params:
    - name: app_path
      type: string
      default: "."
  workspaces:
    - name: source
    - name: config
  steps:
    - name: build
      image: registry.example.com/my-image:tag
      workingDir: $(workspaces.source.path)/$(params.app_path)
      env:
        - name: HOME
          value: $(workspaces.source.path)
      script: |
        #!/usr/bin/env bash
        set -e
        echo "doing work"
```

**Ссылки внутри Task:**
- `$(params.<name>)` — параметр
- `$(workspaces.<name>.path)` — путь к смонтированному workspace
- `$(workspaces.<name>.path)` доступен только если workspace объявлен в `spec.workspaces` этой Task

---

## 2. TaskRun — разовый запуск Task (редко используется напрямую на экзамене)

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: my-taskrun
spec:
  taskRef:
    name: my-task
  params:
    - name: app_path
      value: "demo-app"
  workspaces:
    - name: source
      persistentVolumeClaim:
        claimName: my-pvc
```

---

## 3. Pipeline — цепочка Tasks

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: my-pipeline
spec:
  params:
    - name: GIT_REPO
      type: string
    - name: APP_PATH
      type: string
      default: "."
  workspaces:
    - name: shared
    - name: maven-config
  tasks:

    # Вариант А: ссылка на существующий Task
    - name: clone-repository
      taskRef:
        name: git-clone        # без "kind: ClusterTask" — их больше нет в новых версиях!
      params:
        - name: url
          value: $(params.GIT_REPO)
        - name: revision
          value: "main"
        - name: deleteExisting
          value: "true"
      workspaces:
        - name: output          # имя, которое ожидает САМА Task git-clone
          workspace: shared      # имя workspace на уровне Pipeline

    # Вариант Б: inline Task прямо внутри Pipeline (taskSpec)
    - name: maven-build
      runAfter:
        - clone-repository
      taskSpec:
        params:
          - name: app_path
            type: string
        workspaces:
          - name: source
          - name: maven-config
        steps:
          - name: build
            image: ""            # <-- частый "пропуск" на экзамене, нужно заполнить
            workingDir: $(workspaces.source.path)/$(params.app_path)
            script: |
              mvn clean package -s $(workspaces.maven-config.path)/settings.xml
      params:
        - name: app_path
          value: $(params.APP_PATH)
      workspaces:
        - name: source           # имя параметра внутри taskSpec
          workspace: shared       # имя workspace на уровне Pipeline
        - name: maven-config
          workspace: maven-config
```

**Запомнить про workspaces:**
- `name:` — как Task НАЗЫВАЕТ workspace у себя внутри (смотрит "внутрь")
- `workspace:` — как Pipeline называет тот же физический диск глобально (смотрит "наружу")
- Это как параметр функции (`name`) и аргумент при вызове (`workspace`)

---

## 4. PipelineRun — запуск Pipeline с реальными данными

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: my-pipeline-run-     # generateName -> можно запускать много раз
spec:
  pipelineRef:
    name: my-pipeline
  params:
    - name: GIT_REPO
      value: "https://github.com/user/repo.git"
  taskRunTemplate:
    serviceAccountName: pipeline
    podTemplate:
      securityContext:
        runAsUser: 65532          # фиксируем UID, если разные Task конфликтуют по правам
        fsGroup: 65532
  workspaces:
    - name: shared
      persistentVolumeClaim:
        claimName: my-pvc          # готовый PVC (статический provisioning)
      # ИЛИ для динамического provisioning:
      # volumeClaimTemplate:
      #   spec:
      #     accessModes: [ReadWriteOnce]
      #     resources:
      #       requests:
      #         storage: 1Gi
    - name: maven-config
      secret:
        secretName: mvn-settings
```

**Запуск через CLI вместо YAML:**
```bash
tkn pipeline start my-pipeline \
  -w name=shared,claimName=my-pvc \
  -w name=maven-config,secret=mvn-settings \
  -p GIT_REPO=https://github.com/user/repo.git \
  --showlog
```

---

## 5. Триггеры — автозапуск PipelineRun по webhook

### Порядок создания (важно соблюдать!)
```
TriggerBinding → TriggerTemplate → Trigger → EventListener → expose svc → URL в webhook
```

### TriggerBinding — извлекает поля из JSON payload webhook'а

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: my-binding
spec:
  params:
    - name: git-repo-url
      value: $(body.repository.url)
    - name: git-repo-name
      value: $(body.repository.name)
    - name: git-revision
      value: $(body.head_commit.id)
```

Частые поля payload (GitHub):
- `$(body.repository.url)`
- `$(body.repository.name)`
- `$(body.head_commit.id)` — SHA коммита
- `$(body.ref)` — ветка (refs/heads/main)

### TriggerTemplate — шаблон PipelineRun

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: my-template
spec:
  params:
    - name: git-repo-url
    - name: git-repo-name
    - name: git-revision
      default: main
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: build-$(tt.params.git-repo-name)-
      spec:
        taskRunTemplate:
          serviceAccountName: pipeline
        pipelineRef:
          name: my-pipeline
        params:
          - name: GIT_REPO
            value: $(tt.params.git-repo-url)
          - name: GIT_REVISION
            value: $(tt.params.git-revision)
        workspaces:
          - name: shared
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 500Mi
```

Внутри TriggerTemplate ссылки на параметры — через `$(tt.params.<name>)`.

### Trigger — связывает Binding + Template

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: my-trigger
spec:
  taskRunTemplate:
    serviceAccountName: pipeline
  bindings:
    - ref: my-binding
  template:
    ref: my-template
```

### EventListener — точка входа (создаёт Pod + Service)

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: my-listener
spec:
  taskRunTemplate:
    serviceAccountName: pipeline
  triggers:
    - triggerRef: my-trigger
```

### Получить внешний URL и настроить webhook

```bash
oc expose svc el-my-listener
oc get route el-my-listener
# вставить полученный URL в Settings → Webhooks на GitHub/Gitea
```

---

## Полезные команды для диагностики/повторения синтаксиса на экзамене

```bash
# Встроенная документация по полям CRD (работает офлайн!)
oc explain pipeline.spec
oc explain pipeline.spec.tasks
oc explain task.spec.steps
oc explain triggerbinding.spec
oc explain triggertemplate.spec
oc explain eventlistener.spec

# Посмотреть что уже есть в кластере (можно взять как шаблон)
oc get pipeline -A
oc get task -A
oc get triggertemplate -A
oc get pipeline <name> -o yaml > template.yaml

# Список/статус
tkn pipeline list
tkn pipelinerun list
tkn taskrun list
tkn task list

# Логи
tkn pipelinerun logs -f
tkn pipelinerun logs <name> -f -n <namespace>

# Запуск
tkn pipeline start <name> -w name=...,claimName=... -p PARAM=value --showlog
oc create -f run.yaml

# Диагностика упавшего шага
oc describe taskrun <name> -n <namespace>
oc describe pod <pod-name> -n <namespace>
oc get pods -n <namespace>
```

---

## Частые ошибки и их причины (из практики)

| Симптом | Вероятная причина |
|---|---|
| `Required value: spec.steps[0].image` | Пустой/отсутствующий `image:` в step (частый "пропуск" на экзамене) |
| `no PV available` / PVC висит Pending | Нет StorageClass — нужен готовый PVC или ручной PV |
| `Permission denied` при git-clone/mvn | UID разных Task не совпадает; общий workspace создаётся под одним UID, читается под другим |
| `0/1 nodes are available: pod affinity` | Affinity Assistant (coschedule) — на SNO мешает, отключить: `oc patch tektonconfig config --type merge -p '{"spec":{"pipeline":{"coschedule":"disabled"}}}'` |
| `container has runAsNonRoot and image will run as root` | Образ не подходит под SCC restricted — нужен образ, который умеет работать под произвольным non-root UID |
| `Could not create local repository at /home/...` | Образ ожидает домашнюю директорию, недоступную для записи — переопределить `HOME` через `env:` |
| `failed to list ClusterTasks` | ClusterTask удалены в новых версиях — используй namespaced `Task` |
| TriggerBinding не подставляет значения | Имена параметров в Binding не совпадают с именами, которые ожидает TriggerTemplate |

---

## Мысленная модель (если забыл всё остальное — вспомни это)

- **Task** = функция
- **Pipeline** = вызов нескольких функций по очереди (`runAfter` = порядок)
- **PipelineRun** = конкретный вызов с конкретными аргументами
- **workspace** = общая папка между контейнерами (как `docker -v host:container`)
- **TriggerBinding** = "что вытащить из JSON" / **TriggerTemplate** = "что с этим сделать" / **Trigger** = склейка двух / **EventListener** = HTTP-сервер, который всё это запускает
