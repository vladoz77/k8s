Pod [lifecycle hooks](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) запускают события на определенных этапах жизненного цикла кластера. Например, если происходит завершение работы Pod, вы можете создать сценарий, который сообщит вам об изящном завершении контейнера перед его завершением, или сценарий, который сообщит вам о запуске **initContainer** перед запуском основного контейнера приложения.

Роль hook-ов жизненного цикла заключается в настройке правильных обработчиков для идеального управления вашими рабочими нагрузками. Например, балансировщик нагрузки может по-прежнему отправлять трафик на Pods, который уже находится в завершении. Lifecycle hooks справляются с такими ситуациями, выполняя асинхронные задачи, которые запускают инициализацию и процесс снятия с регистрации ваших контейнеров. Они проверяют выполнение этих задач и затем переходят к запуску контейнера Pod или его завершению. Kubernetes предоставляет контейнерам следующие крючки жизненного цикла:

- **postStart**: Его обработчики отправляются сразу после запуска контейнера для выполнения задач инициализации.
- **preStop**: Его события выполняются непосредственно перед завершением работы контейнера. Они выполняют такие задачи, как очистка и изящное завершение работы контейнера перед его завершением.

Kubernetes предоставляет следующие два подхода для реализации этих [hook handlers](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#hook-handler-implementations):

- **Exec** выполняет указанную команду внутри контейнера.
- **HTTP** создает обработчики HTTP-запросов к URL, запущенному внутри контейнера.

### Использование хуков жизненного цикла

Эти хуки применяются только к контейнеру, в котором они определены. Давайте создадим несколько манифестов Kubernetes, чтобы продемонстрировать, как работает механизм хуков. Ниже приведен простой пример выполнения базового хука `postStart`:

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: testcontainer
    image: nginx
    # Спецификация lifecycle hook 
    lifecycle:
      postStart:
      # Выполним следущую команду, которая создат файл /test.log
        exec:
          command: ["/bin/sh", "-c",  "echo postStart hook executed > /test.log"]
```

Чтобы протестировать хук `postStart`, используйте `kubectl` для запуска оболочки, которая получает доступ к запущенному контейнеру Pods:

```Bash
kubectl exec --stdin --tty test-pod -- sh
```

Теперь вы можете прочитать содержимое, записанное в файл **test.log**, следующим образом:

```Bash
cat /test.log 

postStart hook executed
```

Теперь вы можете прочитать содержимое, записанное в файл **test.log**, следующим образом:

### Использование хука PreStop

Хук preStop выполняет команду непосредственно перед завершением работы контейнера. Посмотрите следующий пример, в котором команда **redis-cli shutdown** выполняется с помощью хука `preStop`.

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: redis-pod
spec:
  containers:
  - name: redis-container
    image: redis
    ports:
    - containerPort: 6379
    volumeMounts:
    - name: redis-data
      mountPath: /src/redis/data
    lifecycle:
      preStop:
        exec:
          command: ["redis-cli shutdown"]
  volumes:
  - name: redis-data
    emptyDir: {}
```

Чтобы узнать, как работает `preStop` в этом сценарии, проверьте логи контейнера **redis-container**, созданного выше:

```YAML
kubectl logs -f redis-pod -c redis-container
```

Попробуйте завершить работу **redis-pod**, наблюдая за логами вашего контейнера в приведенном выше выводе терминала:

```YAML
kubectl delete pod/redis-pod
```

Когда вы удалите  Pod, ваш preStop будет выполнен до завершения работы Pod. Kubernetes отправляет сигнал [TERM](https://www.i-programmer.info/programming/cloud/15652-how-to-terminate-a-process-gracefully-with-sigterm-in-kubernetes-.html#:~:text=SIGTERM%20Signal%20%2D%20Kubernetes%20will%20then,all%20data%20while%20it%20does.) главному процессу Pod, чтобы позволить всем контейнерам изящно завершить работу и завершение всех дочерних процессов.

```bash
1:signal-handler (1744810782) Received SIGTERM scheduling shutdown...
1:M 16 Apr 2025 13:39:42.369 * User requested shutdown...
1:M 16 Apr 2025 13:39:42.369 * Saving the final RDB snapshot before exiting.
1:M 16 Apr 2025 13:39:42.398 * DB saved on disk
1:M 16 Apr 2025 13:39:42.399 # Redis is now ready to exit, bye bye...
```
## Лучшие практики управления жизненным циклом Pods

Жизненный цикл Pods - это важная концепция управления кластером Kubernetes. Чтобы обеспечить бесперебойную работу Pods, необходимо придерживаться следующих правил:

- Используйте liveness и readiness [container probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes), чтобы определить, правильно ли работает контейнер в Pod и готов ли он принимать трафик.
- Настройте [restartPolicy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions) для обеспечения автоматического перезапуска контейнеров в случае их сбоя.
- Создайте хуки жизненного цикла [container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) для выполнения задач инициализации, завершения и очистки.
- Следите за работой Pods, чтобы убедиться в их правильном функционировании, а также для обнаружения и устранения неполадок.
- Проектируйте высокую доступность с помощью таких инструментов, как` replica sets` и `deployments`, чтобы ваши приложения автоматически [масштабировались](https://loft.sh/blog/what-does-it-mean-to-scale-a-deployment/?utm_medium=reader&utm_source=loft-blog&utm_campaign=blog_an-introductory-guide-to-managing-the-kubernetes-pods-lifecycle) и могли восстанавливаться после сбоев.
- Используйте [ Pod garbage collector (PodGC)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes), чтобы избежать утечки ресурсов при создании и завершении работы Pods.
