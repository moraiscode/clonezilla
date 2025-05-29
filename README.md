# Backup Automático com Clonezilla para SSD de Imagem

Este guia mostra como automatizar a criação de uma **imagem completa do SSD principal** com Clonezilla, salvando-a em um segundo SSD, para fins de restauração em caso de falha.

---

## 🧩 Objetivo

- Clonar automaticamente o SSD principal (ex: /dev/sda)
- Salvar a imagem no SSD de backup (ex: /dev/sdb1)
- Executar o processo diariamente às 2h da manhã
- Permitir restauração futura para outro disco, com todos os dados intactos

---

## 🔍 Etapa 1 – Identificar os Discos

Use o comando:

    lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

Exemplo de saída:

    NAME   SIZE TYPE MOUNTPOINT
    sda    500G disk
    ├─sda1 512M part /boot/efi
    ├─sda2 100G part /
    ├─sda3 400G part /home
    sdb    500G disk
    └─sdb1 500G part

Verifique qual é o SSD de origem (sistema) e o SSD de backup.

---

## 📝 Etapa 2 – Criar Script de Backup

Criar o script:

    sudo nano /usr/local/bin/clonar_para_imagem.sh

Conteúdo do script:

    #!/bin/bash

    # Variáveis
    DATA=$(date +'%Y-%m-%d_%H-%M')
    MOUNT_POINT="/mnt/backup"
    DEVICE="/dev/sdb1"
    IMAGEM="$DATA-clone"
    LOGFILE="/var/log/clonezilla/clonar_$DATA.log"

    # Criar diretório de log
    mkdir -p /var/log/clonezilla

    # Montar SSD de backup se ainda não estiver montado
    mount | grep "$MOUNT_POINT" > /dev/null
    if [ $? -ne 0 ]; then
      sudo mount $DEVICE $MOUNT_POINT
    fi

    # Criar diretório para imagens se necessário
    mkdir -p $MOUNT_POINT/clonezilla

    # Executar Clonezilla para salvar o disco sda
    sudo ocs-sr -q2 -j2 -z1p -i 2000 -sfsck -scr -senc -nogui savedisk "$IMAGEM" sda \
      --target-dir "$MOUNT_POINT/clonezilla" >> $LOGFILE 2>&1

    # Desmontar após o backup
    sudo umount $MOUNT_POINT

Tornar o script executável:

    sudo chmod +x /usr/local/bin/clonar_para_imagem.sh

---

## ⏰ Etapa 3 – Agendar no Cron

Abra o crontab do root:

    sudo crontab -e

Adicione a seguinte linha:

    0 2 * * * /usr/local/bin/clonar_para_imagem.sh

Isso agendará a execução do script todos os dias às 2h da manhã.

---

## ♻️ Etapa 4 – Restaurar o Sistema

Se for necessário restaurar o sistema após falha do SSD:

1. Bootar com Clonezilla Live
2. Montar o SSD de backup (ex: /dev/sdb1)
3. Escolher a imagem salva (ex: 2025-05-29_02-00-clone)
4. Restaurar para o novo SSD (/dev/sda)

---

## 📦 Etapa 5 – (Opcional) Compactar a Imagem como .tar.gz

Caso deseje compactar a imagem para facilitar o transporte/backup externo:

    cd /mnt/backup/clonezilla/
    tar -cvzf 2025-05-29-clone.tar.gz 2025-05-29_02-00-clone/

---

## 🧠 Dicas Finais

- Nunca confunda sda e sdb — isso pode apagar seu sistema!
- Certifique-se de que o computador esteja ligado às 2h para o cron rodar
- Você pode programar um find para excluir backups antigos automaticamente

---

## ✅ Requisitos

- Clonezilla instalado: sudo apt install clonezilla
- SSD de backup com partição montável (ex: ext4)
- Permissão de root para montar e executar o script

---

## 🛠️ Possíveis Extensões

- Notificação por e-mail após o backup
- Manter somente os últimos 7 backups
- Criar .iso bootável a partir da imagem
