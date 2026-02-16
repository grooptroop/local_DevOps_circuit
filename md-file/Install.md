
Этапы установки:

1. [k3s кластер](#k3s-кластер)

2. [MetalLB](#metallb)

3. [Docker-registry](#docker-registry)

4. [Gitea](#gitea)

5. [Drone-CI](#drone-ci)

---

## k3s кластер

Чтобы k3s функционировала нам необходимо включить на сервере cgroup контроллеры 

1. cgroup_enable=cpuset (возможность привязывать контейнеры к CPU-ядрам)
2. cgroup_enable=memory (отвечает за лимиты и учёт потребления RAM)
3. cgroup_memory=1 (использование v1 или смешанного режима памяти)
```
sudo nano /etc/default/grub
``` 
Затем ребутаем 
```
sudo reboot
```

На всякий случай напоминаю что `curl` не является встроенным инструментом
```
sudo apt install curl
```

Скачиваем с зеркала gitHub потому что огранечения в Российском регионе
```
curl -L https://github.com/k3s-io/k3s/releases/download/v1.30.4+k3s1/k3s-arm64 -o /usr/local/bin/
sudo chmod +x /usr/local/bin/k3s
```

Ручками запускаем сервис
```
sudo tee /etc/systemd/system/k3s.service >/dev/null <<'EOF'
[Unit]
Description=k3s
After=network.target
[Service]
Type=notify
ExecStart=/usr/local/bin/k3s server --disable servicelb # для дальнейшей установки MetalLB
TimeoutStartSec=0
Restart=always
RestartSec=10
KillMode=mixed
LimitNOFILE=65536
LimitNPROC=infinity
TasksMax=infinity
OOMScoreAdjust=-1000
[Install]
WantedBy=multi-user.target
EOF
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now k3s
```

Проверим что всё стоит
```
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

Копируем токен для подключения к k3s кластеру
```
sudo cat /var/lib/rancher/k3s/server/node-token
```

На 2 сервере ставим 
```
curl -L https://github.com/k3s-io/k3s/releases/latest/download/k3s-arm64 -o k3s
chmod +x k3s
sudo mv k3s /usr/local/bin/
```

```
sudo k3s agent --server https://<ip_or_hostname_of_existing_node>:6443 --token <server_token>
```

---

## MetalLB

Скачиваем MetalLB

Скачиваем Helm если нет 
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

создаём симлинк kubectl - k3s (shell запускает /usr/local/bin/kubectl, а это на самом деле ссылка на /usr/local/bin/k3s)
```
sudo rm /usr/local/bin/kubectl
sudo ln -s /usr/local/bin/k3s /usr/local/bin/kubectl
```

для helm дополнительно прописываем
```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Устанавливаем Helm
```
helm install metallb metallb/metallb -n metallb-system --create-namespace
```

Берём yaml в котором прописан набор доступных для MetalLb ip-адресов
```
cd ~ 
curl -L -o metalLB-pool.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/MetalLB/metallb-pool.yaml"
``` 

Подправим ip адреса если нам нужно
```
nano metalLB-pool.yaml
```

Запускаем yaml файл
```
kuberctl apply -f metalLB-pool.yaml
```


!Если у нас падаёт всё в metalLB удаляем поды, они встанут сразу норм
```
kubectl delete pod -n metallb-system -l app.kubernetes.io/component=controller
kubectl delete pod -n metallb-system -l app.kubernetes.io/component=speaker
```

---

## Docker-registry

Создаём пространство имён
```
Kubectl create namespace docker-registry
```

Берём yaml 
```
cd ~ 
curl -L -o docker-registry.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Docker-registry/docker-registry.yaml"
```


чтобы любой узел мог ходить к registry по IP master’а.
```
sudo mkdir -p /etc/rancher/k3s
curl -L -o registries.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Docker-registry/registries.yaml"
sudo mv registries.yaml /etc/rancher/k3s/
```

Пкркзапускаем k3s
```
sudo systemctl restart k3s 
```

Повторяем тоже самое на агенте
```
sudo mkdir -p /etc/rancher/k3s
curl -L -o registries.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Docker-registry/registries.yaml"
sudo mv registries.yaml /etc/rancher/k3s/
```

Перезапуск Агента
```
sudo systemctl restart k3s-agent || sudo systemctl restart k3s
```

Переходим на агента и прописываем адрес insecure-registry
```
sudo nano /etc/docker/daemon.json
```

```
{
  "insecure-registries": ["192.168.1.70:30500"]
}
```

Ставим докер если его нет и создаём образ который пушим в наш registry
```
docker pull nginx:alpine
docker tag nginx:alpine 192.168.1.70:30500/our-task/nginx:v1
docker push 192.168.1.70:30500/our-task/nginx:v1
```

Проверяем появился ли образ:
```
curl http://192.168.1.70:30500/v2/_catalog
```
Если всё работает, ура мы сделали! 

Если у нас не ставятся образы:

Создайём storage
```
sudo mkdir -p /var/lib/rancher/k3s/storage
sudo chown -R 1001:65533 /var/lib/rancher/k3s/storage/
sudo chmod 755 /var/lib/rancher/k3s/storage/
```

перезапустим поды
```
kubectl delete pod docker-registry-755c67db65-89rc8 -n docker-registry
```

ждём запуска
```
kubectl get pod -n docker-registry
```

Собираем образ из каталога
```
cd ~ 
curl -L -o our-task-deploy.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Docker-registry/our-task-deploy.yaml"
``` 

Если запустился, значит образ встал из каталога
```
kubectl apply -f our-task-deploy.yaml

kubectl get pods,deploy -n default
```

---

## Gitea


Создадим namespace для Gitea
```
kubectl create namespace gitea
```

Устанавливаем Gitea через yaml 
```
cd ~ 
curl -L -o gitea.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Gitea/gitea.yaml"
```

Открываем её в браузере(как указали в gitea.yaml) 
```
http://192.168.1.201:3000/
```

---

## Drone-CI

Заходим в gitea в раздел приложения
```
http://192.168.1.201:3000/user/settings/applications
```

Здесь скролим вниз и выбираем создать OAuth2 приложения

Вставляем данные:

Имя `Drone-CI`

Redirect URL `http://192.168.1.202:80/login`

Сохраняем и выписываем:

- Client ID
- Client Secret

На любой нашей ноде создаём секрет для Drone (тоже сохраняем себе)
```
openssl rand -hex 16
```

Создаём namespaces
```
kubectl create namespace drone
```

Скачиваем yaml для Drone-CI и подставляем затем наши данные(я подписал куда что вставлять)
```
cd ~ 
mkdir Drone-CI  && cd Drone-CI
curl -L -o drone-server.yaml "https://raw.githubusercontent.com/grooptroop/homelab-devops-platform/refs/heads/master/Drone-ci/drone-server.yaml"
```

Применяем и смотрим всё ли поднялось
```
kubectl apply -f drone-server.yaml
kubectl get pods,svc -n drone
```

Заходим на наш адрес Drone-CI разрешаем доступ и регистрируемся 
```
http://192.168.1.202
```

Начинаем настраивать drone-runner
```
cd Drone-CI
curl -L -o drone-runner.yaml " "
```