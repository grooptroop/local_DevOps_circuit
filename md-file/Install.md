
Этапы установки:

1. [k3s кластер](#k3s-кластер)

2. [MetalLB](#metallb)

3. [Docker-registry](#docker-registry)



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
curl -L -o registries.yaml ""
sudo mv registries.yaml /etc/rancher/k3s/
```

Пкркзапускаем k3s
```
sudo systemctl restart k3s 
```

Повторяем тоже самое на агенте
```
sudo mkdir -p /etc/rancher/k3s
curl -L -o registries.yaml ""
sudo mv registries.yaml /etc/rancher/k3s/
```

Перезапуск Агента
```
sudo systemctl restart k3s-agent || sudo systemctl restart k3s
```