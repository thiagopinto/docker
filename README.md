# Docker

## Redis

A configuração em [redis/](redis/) usa Redis 8.10.3, com autenticação por
ACL e TLS mútuo obrigatório. A versão da imagem é fixada e deve ser
atualizada periodicamente para receber correções de segurança. A porta é
publicada somente em `127.0.0.1`;
outros contêineres na mesma rede Docker ainda podem alcançá-la e devem ser
tratados como não confiáveis. O sistema de arquivos raiz é somente leitura,
as capacidades Linux são removidas e os dados permanecem em `redis/data/`.
Não há senha nem certificado padrão: a instância não inicia sem os arquivos
necessários.

### Migração

Antes de atualizar, pare o Redis antigo e faça uma cópia consistente de
**todo** o diretório `redis/data/`, incluindo arquivos RDB e AOF. Confira a
compatibilidade da versão antiga e teste a restauração em uma cópia antes de
atualizar a produção. Não execute duas instâncias sobre o mesmo diretório.
Quando existentes, arquivos AOF têm prioridade sobre snapshots RDB. Após a
atualização, não presuma que uma versão antiga consiga ler os novos arquivos:
o rollback exige o backup anterior. A configuração habilita AOF com
`appendfsync everysec`, que pode perder aproximadamente um segundo de escritas
em uma falha; mantenha backups externos e teste sua restauração.

### Credenciais e certificados

Em `redis/`, crie `data/` e `secrets/` e forneça os seguintes arquivos:

* `secrets/ca.crt`: CA que valida os certificados dos clientes.
* `secrets/server.crt` e `secrets/server.key`: certificado e chave privada
  do servidor. O certificado deve identificar o nome ou IP usado pelo cliente
  e ser assinado por uma CA confiável para ele.
* `secrets/redis.acl`: ACL descrita abaixo.

O cliente também precisa de um certificado e chave privados válidos perante
a CA configurada no servidor, além de confiar na CA do certificado do
servidor. Em produção, use certificados emitidos pela sua infraestrutura de
PKI, proteja as chaves privadas e acompanhe sua expiração e rotação. Não coloque
a chave privada da CA no projeto nem no contêiner.

Exemplo de criação de uma senha aleatória e da ACL, executado em `redis/`:

```sh
mkdir -p data secrets
chmod 755 secrets
umask 077
openssl rand -hex 32 | tr -d '\n' > secrets/app.password
hash=$(openssl dgst -sha256 -r < secrets/app.password | cut -d ' ' -f 1)
printf 'user default off\nuser app on #%s ~* +@read +@write +@connection +@transaction -@dangerous\n' "$hash" > secrets/redis.acl
unset hash
```

O usuário `app` é um ponto de partida: pode acessar todas as chaves e executar
comandos nas categorias listadas, exceto os classificados como perigosos.
Restrinja os comandos e padrões de chaves às necessidades reais de cada
aplicação; crie usuários separados quando necessário. Ele não é administrador
e não possui acesso a todos os comandos ou canais Pub/Sub. A ACL é montada
somente para leitura. Depois de alterar a ACL ou substituir certificados,
execute `docker compose up -d --force-recreate` em `redis/` para remontar os
arquivos e aplicar as mudanças. Proteja também `app.password`, usada pelo
cliente e nunca montada no contêiner.

A imagem executa Redis como UID/GID `999:999`. Com Docker no Linux sem
remapeamento de usuários, ajuste a propriedade e as permissões para que esse
usuário possa ler os arquivos e gravar em `data/`:

```sh
sudo chown -R 999:999 data
sudo chown 999:999 secrets/redis.acl secrets/ca.crt secrets/server.crt secrets/server.key
sudo chmod 700 data
sudo chmod 600 secrets/redis.acl secrets/server.key
sudo chmod 644 secrets/ca.crt secrets/server.crt
```

Em Docker rootless, remapeamento de usuários ou SELinux, adapte os IDs e os
rótulos de acesso ao ambiente; não torne as chaves privadas públicas. Os
arquivos em `secrets/` e os dados são ignorados pelo Git e pelo contexto de
build, mas ainda precisam de proteção e backup no host.

### Inicialização e conexão

```sh
cd redis
docker compose up -d --build
docker compose logs redis
```

Com `redis-cli` instalado no host, use um certificado de cliente válido e a
senha criada acima (ajuste os caminhos dos arquivos de cliente). O exemplo
pressupõe que a mesma CA assina os certificados do servidor e do cliente;
caso contrário, passe a CA do servidor em `--cacert`.

```sh
REDISCLI_AUTH="$(cat secrets/app.password)" redis-cli --tls \
  --cacert secrets/ca.crt --cert /caminho/client.crt --key /caminho/client.key \
  --user app -h 127.0.0.1 -p 6379 PING
```

O resultado esperado é `PONG`. Os clientes existentes precisam ser
configurados para TLS mútuo e autenticação com usuário e senha. Não exponha a
porta externamente sem rever a arquitetura, firewall e controles de rede.
Esta configuração não fornece alta disponibilidade, monitoramento ou backup
automatizado; esses controles devem ser planejados separadamente para produção.
