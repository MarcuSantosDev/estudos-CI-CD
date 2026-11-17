### Setup inicial no servidor (feito uma vez, manualmente)

## Criando um usuário de deploy no servidor

- **Nunca use o usuário root pra deploy**
- Cria um usuário específico com permissões limitadas:

```bash
# no servidor, como root
adduser deploy

# troca pro usuário deploy
su - deploy
```

- Isola o acesso — se a chave SSH vazar, o atacante só tem acesso ao usuário `deploy`, não ao root

---

## Configurando autenticação SSH sem senha

- O pipeline precisa conectar no servidor sem digitar senha — usa par de chaves SSH

### No servidor (como usuário deploy):

```bash
# cria o diretório .ssh se não existir
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# cria o arquivo authorized_keys se não existir
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Na sua máquina local:

#### Se for Linux / macOS:
```bash
# gera um par de chaves específico pra deploy
ssh-keygen -t ed25519 -C "deploy@github-actions" -f ~/.ssh/github_actions_deploy

# isso gera dois arquivos:
# ~/.ssh/github_actions_deploy      ← chave privada (vai pro GitHub Secrets)
# ~/.ssh/github_actions_deploy.pub  ← chave pública (vai pro servidor)
```

#### Se for Windows:
```bash
# Usando PowerShell (recomendado)
ssh-keygen -t ed25519 -C "deploy@github-actions" -f "$HOME\.ssh\github_actions_deploy"

# Ou usando Git Bash (mesmo comando do Linux):
ssh-keygen -t ed25519 -C "deploy@github-actions" -f ~/.ssh/github_actions_deploy
```

### Copia a chave pública pro servidor:

#### Se for Linux / macOS:
```bash
# adiciona a chave pública ao authorized_keys do servidor
cat ~/.ssh/github_actions_deploy.pub | ssh deploy@SEU_IP "cat >> ~/.ssh/authorized_keys"
```

#### Se for Windows:
```bash
# Opção 1: Usando PowerShell
Get-Content "$HOME\.ssh\github_actions_deploy.pub" | ssh deploy@SEU_IP "cat >> ~/.ssh/authorized_keys"

# Opção 2: Usando scp (mais confiável se o pipe falhar)
scp "$HOME\.ssh\github_actions_deploy.pub" deploy@SEU_IP:~/temp.pub
ssh deploy@SEU_IP "cat ~/temp.pub >> ~/.ssh/authorized_keys && rm ~/temp.pub"
```


### Testa a conexão:

#### No Linux / macOS / Git bash:
```bash
ssh -i ~/.ssh/github_actions_deploy deploy@SEU_IP
```

#### No Windows (PowerShell):
```bash
ssh -i "$HOME\.ssh\github_actions_deploy" deploy@SEU_IP
```




### Configurando Node e PM2 (com usuário deploy)

```bash
# instala o Node via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
# teste se instalou:
nvm -v

# se não funcionou, provavelmente precisa sair e entrar no usuário com:
exit
su - deploy

# se funcionou, continua:
nvm install 22
nvm use 22

# instala o PM2 globalmente
npm install -g pm2

# clona o repositório
git clone https://github.com/SEU_USER/NOME_DO_REPO.git /home/deploy/app
cd /home/deploy/app

# instala dependências e faz o build inicial
npm install
npm run build

# sobe a aplicação com PM2
pm2 start dist/index.js --name "api"
pm2 save
pm2 startup
```

- **Destaca:** esse setup manual acontece só na primeira vez. A partir daqui, o pipeline assume.

---

