# Bitsong Node Installation Guide
##### Chain ID: bitsong-2b | Current Node Version: v0.15.0

Официальные документы по установке и настройке ноды https://docs.bitsong.io/blockchain/install-go-bitsong

soon...

!! Перед тем как начать установку узла необходимо [подготовить систему](https://github.com/Digital-Freedom-Validator/Node-Installation-Guide/blob/main/!%20Preparing%20the%20system%20by%20reinstalling%20the%20node_ru.md)

## Установка узла
1. Копируем репозиторий
   ```
   git clone https://github.com/bitsongofficial/go-bitsong
   ```
2. Проверяем, что появилась папка go-bitsong
   ```
   ll
   ```
3. Переходим в эту папку
   ```
   cd go-bitsong
   ```
4. Установливаем v0.15.0
   ```
   git checkout v0.15.0
   ```
5. Собираем бинарник
   ```
   make install
   ```
6. Проверяем версию
   ```
   bitsongd version
   ```

## Настройка узла (Присоединяйтесь к основной сети)
1. Инициализируем узел
   ```
   bitsongd init <YOUR_MONIKER> --chain-id bitsong-2b
   ```
2. Скачиваем Genesis файл
   ```
   wget -O ~/.bitsongd/config/genesis.json https://raw.githubusercontent.com/bitsongofficial/networks/master/bitsong-2b/genesis.json
   ```
3. Проверяем состояние валидатора на начальном этапе
   ```
   cd && cat .archway/data/priv_validator_state.json
   ```
   вывод
   {
     "height": "0",
     "round": 0,
     "step": 0
   }

   #если нет, то выполняем команду
   ```
   bitsongd unsafe-reset-all
   ```
4. Правим конфиг ~/.bitsongd/config/client.toml
   ```
   bitsongd config chain-id bitsong-2b
   ```
5. Настраиваем минимальную цену за газ
   ```
   nano .bitsongd/config/app.toml
   ```
   minimum-gas-prices = "0.0025ubtsg"
6. Добавляем сиды и пиры. Я беру актуальные на [Polkachu](https://polkachu.com/live_peers/bitsong)
  
## Запуск узла
1. Устанавливаем Cosmovisor
   ```
   go install github.com/cosmos/cosmos-sdk/cosmovisor/cmd/cosmovisor@v1.0.0
   ```
3. Настраиваем Cosmovisor  
   ```
   mkdir -p ~/.bitsongd/cosmovisor/genesis/bin
   ```  
   ```
   mkdir -p ~/.bitsongd/cosmovisor/upgrades
   ```  
   ```
   cp ~/go/bin/bitsongd ~/.bitsongd/cosmovisor/genesis/bin
   ```
4. Создаем системный файл для работы узла в фоновом процессе с автоматическим перезапуском
   ```
   sudo tee /etc/systemd/system/bitsongd.service > /dev/null <<EOF  
   [Unit]
   Description="bitsong node"
   After=network-online.target

   [Service]
   User=USER
   ExecStart=/home/USER/go/bin/cosmovisor start
   Restart=always
   RestartSec=3
   LimitNOFILE=4096
   Environment="DAEMON_NAME=bitsongd"
   Environment="DAEMON_HOME=/home/USER/.bitsongd"
   Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"
   Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
   Environment="UNSAFE_SKIP_BACKUP=true"

   [Install]
   WantedBy=multi-user.target
   EOF
   ```
   Иногда нужно прописывать с run ```ExecStart=/home/USER/go/bin/cosmovisor start```  
5. Настраиваем демон  
   ```
   sudo -S systemctl daemon-reload  
   sudo -S systemctl enable bitsongd
   ```
6. Запускаем процесс и подтверждаем, что он запущен
   ```
   sudo -S systemctl start bitsongd
   sudo service bitsongd status
   ```
7. Синхронизируем узел с текущим состоянием блокчейна
   ```
   sudo systemctl stop bitsong

   SNAP_RPC="https://rpc.bitsong.forbole.com:443"
   SNAP_RPC2="https://bitsong.stakesystems.io:2053"
   
   LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
   BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000)); \
   TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

   peers="e2b9971222adf71f7199c670b7e85471c447e926@157.90.255.143:26656,120740c15a8a19c232b1aa4d80b20de248b33db3@135.181.129.94:26656,bbfb37b3c44c8148b6af7adfa016ec8fabff69d1@121.78.247.243:16656,d741773bc5eecbefb7b14fcca5e3e0fedd49d5a3@157.90.95.104:26656,6e93a30587671e2cecacbcbb27092809bb20249f@95.217.203.59:31656,adfe1cf240780cf8d58266171ced72fb4e9a7a6d@23.226.14.168:26656,f36d3a926ae0583e60f00e7bc54711f3cb7fe769@195.201.58.166:26656,9c9f030298bdda9ca69de7db8e9a3aef33972fba@135.181.16.236:31656,9806602afb65ba45d1048d65285d5c6e50285088@178.18.242.242:26656,4fdd438ea70927003022ecc308e36bc1924ec598@51.210.104.207:26656,3cf3effd3ecb33bdbb5c5e6528c88fde4869b97c@116.202.139.113:26656,2cd6bb75fc9279c62c0ef3af82fbe08632743472@bitsong-peer.panthea.eu:31656"
  
   sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.bitsongd/config/config.toml
     
   sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
   s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC2\"| ; \
   s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
   s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.bitsongd/config/config.toml
     
   cp $HOME/.bitsongd/data/priv_validator_state.json $HOME/.bitsongd/priv_validator_state.json.backup

   bitsongd tendermint unsafe-reset-all --keep-addr-book --home "$HOME/.bitsongd"

   mv $HOME/.bitsongd/priv_validator_state.json.backup $HOME/.bitsongd/data/priv_validator_state.json

   sudo systemctl start bitsong

   sudo journalctl -u bitsong -f
8. Проверяем высоту сети  
   ```
   bitsongd status 2>&1 | jq ."SyncInfo"."latest_block_height"
   ```

   ## Создание валидатора
   
