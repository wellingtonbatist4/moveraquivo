# moveraquivo

#!/bin/bash

LOG_FILE="/var/log/migracao_arquivos.log"
MAX_ITERACOES=50000
MIGRADOS=0
IGNORADOS_EXISTE=0
IGNORADOS_PERM=0
contador=0

DESTINO_BASE="/bucket"

log() {
    echo "$(date) - $1" | tee -a "$LOG_FILE"
}

log "Iniciando processo de migração"

# Lendo o mapa de IPs para hostnames
declare -A HOST_MAP
if [[ -f /etc/host_map.conf ]]; then
    log "Lendo mapeamento de IP para hostname..."
    while IFS='=' read -r ip hostname; do
        [[ -n "$ip" && -n "$hostname" ]] && HOST_MAP["$ip"]="$hostname"
        log "Mapeado: $ip -> $hostname"
    done < /etc/host_map.conf
else
    log "Erro: Arquivo /etc/host_map.conf não encontrado!"
    exit 1
fi

# Obtendo o IP da máquina para identificar hostname correto
IP_LOCAL=$(hostname -I | awk '{print $1}')
log "IP Local obtido: $IP_LOCAL"

if [[ -n "$IP_LOCAL" && -n "${HOST_MAP[$IP_LOCAL]}" ]]; then
    HOSTNAME_LOCAL="${HOST_MAP[$IP_LOCAL]}"
    log "Hostname identificado: $HOSTNAME_LOCAL para IP $IP_LOCAL"
else
    HOSTNAME_LOCAL=$(hostname) # Fallback para o hostname da máquina
    log "IP $IP_LOCAL não encontrado no host_map.conf. Usando hostname da máquina: '$HOSTNAME_LOCAL'."
fi

# Lendo diretórios de origem
if [[ ! -f /etc/migracao_dirs.conf ]]; then
    log "Erro: Arquivo de configuração /etc/migracao_dirs.conf não encontrado!"
    exit 1
fi

if [[ ! -s /etc/migracao_dirs.conf ]]; then
    log "Erro: Arquivo de configuração /etc/migracao_dirs.conf está vazio!"
    exit 1
fi

while IFS= read -r dir_origem; do
    [[ -z "$dir_origem" || "$dir_origem" =~ ^# ]] && continue

    log "Processando diretório: $dir_origem"

    if [[ ! -d "$dir_origem" ]]; then
        log "Diretório de origem não encontrado: $dir_origem"
        continue
    fi

    while IFS= read -r arquivo; do
        (( contador >= MAX_ITERACOES )) && log "Limite máximo atingido." && break

        ((contador++))
        nome_arquivo=$(basename "$arquivo")

        # Obtém o IP de origem do arquivo
        MOUNT_IP=$(df "$arquivo" --output=source | tail -n 1 | grep -oP '(?<=//)[^/]+')
        if [[ -z "$MOUNT_IP" ]]; then
            log "Erro: Não foi possível determinar o IP de origem para $arquivo"
            ((IGNORADOS_PERM++))
            continue
        fi

        # Obtém o hostname de origem do arquivo
        ORIGEM_HOST="${HOST_MAP[$MOUNT_IP]}"
        if [[ -z "$ORIGEM_HOST" ]]; then
            ORIGEM_HOST=$MOUNT_IP # Se não encontrar no mapa, usa o próprio IP
        fi

        # Construção do destino correto
        caminho_relativo="${arquivo#"$dir_origem"}"
        DESTINO="$DESTINO_BASE/$ORIGEM_HOST$caminho_relativo"

        log "Criando diretório: $(dirname "$DESTINO")"
        mkdir -p "$(dirname "$DESTINO")"

        if [[ ! -r "$arquivo" ]]; then
            log "Sem permissão de leitura: $arquivo"
            ((IGNORADOS_PERM++))
            continue
        fi

        if [[ -f "$DESTINO" ]]; then
            log "Arquivo já existe no destino: $DESTINO"
            ((IGNORADOS_EXISTE++))
            continue
        fi

        log "Copiando $arquivo -> $DESTINO"
        if cp -av "$arquivo" "$DESTINO" >> "$LOG_FILE" 2>&1; then
            log "Arquivo migrado com sucesso: $nome_arquivo"
            ((MIGRADOS++))
        else
            log "Erro ao copiar: $nome_arquivo"
        fi

    done < <(find "$dir_origem" -type f -mtime +1095)

done < /etc/migracao_dirs.conf

log "Processo de migração concluído"
log "Arquivos processados: $contador"
log "Arquivos migrados: $MIGRADOS"
log "Ignorados (existentes): $IGNORADOS_EXISTE"
log "Ignorados (sem permissão): $IGNORADOS_PERM"
