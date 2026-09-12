# Agente coletor MedEmerge — instalação

Processo que lê o banco do hospital e envia os indicadores para a plataforma MedEmerge. Abre apenas conexão de **saída** HTTPS — nenhuma porta de entrada, sem VPN.

## Requisitos

- Docker com Docker Compose
- Saída HTTPS liberada para `api.medemerge.com.br`
- Acesso de rede ao banco de dados, a partir desta máquina
- Um usuário de banco **somente leitura**, com acesso apenas às views de BI

## Instalação

1. Clone este repositório na máquina que terá acesso ao banco, e crie o `credentials.yaml` a partir do exemplo:

   ```bash
   git clone https://github.com/MedEmerge/coletor.git
   cd coletor
   cp credentials.example.yaml credentials.yaml
   ```

2. Preencha o `credentials.yaml` com os dois valores enviados pela MedEmerge:

   ```yaml
   public_id: '...'
   agent_secret: '...'
   ```

3. Suba:

   ```bash
   docker compose up -d
   ```

4. Confirme:

   ```bash
   docker compose logs -f
   ```

   Esperado: `config vN recebida`, seguido de `query ... coletada` e `lote ... enviado`.

## Diagnóstico

Um ciclo único, sem entrar em loop — o primeiro comando quando algo não funciona:

```bash
docker compose run --rm coletor --once
```

| Mensagem                           | O que fazer                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| `Faltando public_id, agent_secret` | `credentials.yaml` ausente ou incompleto                                        |
| `é um diretório, não um arquivo`   | O caminho do `credentials.yaml` no compose está errado                          |
| `Credenciais do agente rejeitadas` | Peça um novo `agent_secret` à MedEmerge                                         |
| `ORA-12170` / `TNS:no listener`    | Sem rota até o banco: firewall, rede ou host/porta                              |
| `ORA-01017`                        | Usuário ou senha do banco incorretos — corrigido pela MedEmerge, sem mexer aqui |

Falhas também aparecem no painel da MedEmerge, então o suporte consegue diagnosticar **sem precisar de acesso a esta máquina**.

## Atualização

```bash
# edite a versão da imagem no docker-compose.yml, depois:
docker compose pull && docker compose up -d
```

A fila de reenvio sobrevive à atualização.

## O que fica nesta máquina

Apenas o `credentials.yaml`, com a identidade do agente. **Queries, intervalos e credenciais do banco vêm do servidor** a cada ciclo — alterações não exigem tocar nesta instalação.

A senha do banco existe somente em memória, nunca em disco. Os logs não contêm senha nem dado clínico.

## Sobre a imagem — para a análise de segurança

|                   |                                                         |
| ----------------- | ------------------------------------------------------- |
| Imagem            | `ghcr.io/medemerge/coletor` (GitHub Container Registry) |
| Plataformas       | `linux/amd64` e `linux/arm64`                           |
| Base              | `python:3.12-slim`                                      |
| Usuário           | não-root (uid 10001)                                    |
| Portas expostas   | **nenhuma** — o agente só abre conexões de saída        |
| Destinos de rede  | `api.medemerge.com.br` (HTTPS) e o banco configurado    |
| Bancos suportados | Oracle                                                  |

A imagem embute o **Oracle Instant Client**, necessário para autenticar em bancos Oracle com verificadores de senha mais antigos. Ele é redistribuído sob os _Oracle Free Distribution, Hosting, and Use Terms_, que permitem expressamente a redistribuição sem modificação; a cópia da licença acompanha a imagem em `/opt/oracle/instantclient/BASIC_LITE_LICENSE`.

Os metadados ficam nos labels OCI da imagem:

```bash
docker inspect ghcr.io/medemerge/coletor:1.0.3 --format '{{json .Config.Labels}}'
```

## Suporte

suporte@medemerge.com.br
