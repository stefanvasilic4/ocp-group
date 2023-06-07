# Managing Containers as Services

```
sudo yum module install container-tools

podman login registry.example.com

skopeo inspect docker://registry.example.com/rhel8/mariadb-103

mkdir /home/podsvc/db_data

chmod 777 /home/podsvc/db_data

podman run -d --name inventorydb -p 13306:3306 \
           -v /home/podsvc/db_data:/var/lib/mysql/data:Z \
           -e MYSQL_USER=operator1 \
           -e MYSQL_PASSWORD=redhat \
           -e MYSQL_DATABASE=inventory \
           -e MYSQL_ROOT_PASSWORD=redhat \
           registry.example.com/rhel8/mariadb-103:1-86

mkdir -p ~/.config/systemd/user/

cd ~/.config/systemd/user/

podman generate systemd --name inventorydb --files --new

podman stop inventorydb

podman rm inventorydb

systemctl --user daemon-reload

systemctl --user enable --now container-inventorydb.service

loginctl enable-linger
```