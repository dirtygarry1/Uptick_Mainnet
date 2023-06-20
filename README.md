# Uptick_Mainnet

## Подготовка сервера

## обновляем репозитории
`apt update && apt upgrade -y`
## устанавливаем необходимые утилиты
`apt install curl iptables build-essential git wget jq make gcc nano tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y`

# File2Ban
## устанавливаем и копируем конфиг, который будет иметь больший приоритет
`apt install fail2ban -y && \
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local && \
nano /etc/fail2ban/jail.local`
## раскомментировать и добавить свой IP: ignoreip = 127.0.0.1/8 ::1 <ip>
`systemctl restart fail2ban`
## проверяем status 
`systemctl status fail2ban`
## проверяем, какие jails активны (по умолчанию только sshd)
`fail2ban-client status`
## проверяем статистику по sshd
`fail2ban-client status sshd`
## смотрим логи
`tail /var/log/fail2ban.log`
## останавливаем работу и удаляем с автозагрузки
`systemctl stop fail2ban && systemctl disable fail2ban`

# Устанавливаем GO
`ver="1.19.6" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version`

# Новая установка ноды

## Устанавливаем бинарники
`git clone https://github.com/UptickNetwork/uptick && cd uptick
wget https://github.com/UptickNetwork/uptick/releases/download/v0.2.8/uptick-linux-amd64-v0.2.8.tar.gz
tar -zxvf uptick-linux-amd64-v0.2.8.tar.gz
chmod +x ./uptickd
mv $HOME/uptick/uptickd $HOME/go/bin/`

`uptickd version --long | grep -e version -e commit -e build_tags`
> version: v0.2.8
> build_tags: netgo,ledger
> commit: b7ef90210c63737035f8072e084ca5731e0eb14c

## Инициализируем ноду, чтобы создать необходимые файлы конфигурации
`uptickd init *имя ноды* --chain-id uptick_117-1`

## Скачиваем Genesis
`wget -O $HOME/.uptickd/config/genesis.json "https://raw.githubusercontent.com/UptickNetwork/uptick-mainnet/master/uptick_117-1/genesis.json"`

> Проверим генезис
`sha256sum ~/.uptickd/config/genesis.json`
> df80462fed795d877fb1e372175ec66d004056fa0ec98c6c3ed52a6715efc66f

# Настраиваем конфигурацию ноды
## правим конфиг, благодаря чему мы можем больше не использовать флаг chain-id для каждой команды CLI в client.toml
`uptickd config chain-id uptick_117-1`

## при необходимости настраиваем keyring-backend в client.toml 
`uptickd config keyring-backend os`

## настраиваем минимальную цену за газ в app.toml
`sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025auptick\"/;" ~/.uptickd/config/app.toml`

## добавляем seeds/bpeers/peers в config.toml
`external_address=$(wget -qO- eth0.me)`
`sed -i.bak -e "s/^external_address *=.*/external_address = \"$external_address:26656\"/" $HOME/.uptickd/config/config.toml`

`peers="170397e75ca2b0f4e9f3b1bb5d0d23f9b10f01c7@uptick-sentry-1.p2p.brocha.in:30597,c0b33353fb70d8d71dcb9c8848b3b4207bd56951@uptick-sentry-2.p2p.brocha.in:30598,23e76540bea9b6851b92e280d7e0c123a0d49521@uptick-sentry-3.p2p.brocha.in:30599,94b63fddfc78230f51aeb7ac34b9fb86bd042a77@uptick-rpc.p2p.brocha.in:30601,f97a75fb69d3a5fe893dca7c8d238ccc0bd66a8f@uptick.seed.brocha.in:30600,48e7e8ca23b636f124e70092f4ba93f98606f604@54.37.129.164:55056,8ecd3260a19d2b112f6a84e0c091640744ec40c5@185.165.241.20:26656,8e924a598a06e29c9f84a0d68b6149f1524c1819@57.128.109.11:26656,f05733da50967e3955e11665b1901d36291dfaee@65.108.195.30:21656"`
`sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.uptickd/config/config.toml`
`seeds=""`
`sed -i.bak -e "s/^seeds =.*/seeds = \"$seeds\"/" $HOME/.uptickd/config/config.toml`

## при необходимости увеличиваем количество входящих и исходящих пиров для подключения, за исключением постоянных пиров в config.toml
## может помочь при падении ноды, но увеличивает нагрузку
`sed -i 's/max_num_inbound_peers =.*/max_num_inbound_peers = 50/g' $HOME/.uptickd/config/config.toml`
`sed -i 's/max_num_outbound_peers =.*/max_num_outbound_peers = 25/g' $HOME/.uptickd/config/config.toml
`
## настраиваем фильтрацию "плохих" peers
`sed -i -e "s/^filter_peers *=.*/filter_peers = \"true\"/" $HOME/.uptickd/config/config.toml
`
## изменение timeout_commit (просят 2s) 
`sed -i -e "s/^timeout_commit *=.*/timeout_commit = \"2s\"/" $HOME/.uptickd/config/config.toml`

# (ОПЦИОНАЛЬНО) Настраиваем прунинг вapp.toml

`pruning="nothing"`
`pruning_keep_recent="1000"`
`pruning_keep_every="0"`
`pruning_interval="10"`
`sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.uptickd/config/app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.uptickd/config/app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.uptickd/config/app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.uptickd/config/app.toml`

# (ОПЦИОНАЛЬНО) Выкл индексацию вconfig.toml
`indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.uptickd/config/config.toml`

# (ОПЦИОНАЛЬНО) Вкл/выкл снэпшоты вapp.toml
`snapshot_interval=1000 && \
sed -i.bak -e "s/^snapshot-interval *=.*/snapshot-interval = \"$snapshot_interval\"/" ~/.uptickd/config/app.toml`

# Создаем сервисный файл
`tee /etc/systemd/system/uptickd.service > /dev/null <<EOF`

`[Unit]`

`Description=uptickd`

`After=network-online.target`


`[Service]`

`User=$USER`

`ExecStart=$(which uptickd) start`

`Restart=on-failure`

`RestartSec=3`

`LimitNOFILE=65535`


`[Install]`

`WantedBy=multi-user.target`

`EOF`

`systemctl daemon-reload`

`systemctl enable uptickd`

`systemctl restart uptickd && journalctl -u uptickd -f -o cat`

# Создаем валидатора
`uptickd tx staking create-validator \
--chain-id uptick_117-1 \
--commission-rate 0.05 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.1 \
--min-self-delegation "1000000" \
--amount 1000000000000000000auptick \
--pubkey $(uptickd tendermint show-validator) \
--moniker "<name_moniker>" \
--from <name_wallet> \
--fees 5000auptick \
--gas auto`
