# Guia Passo a Passo — Do Zero ao Deploy (Docker Desktop + GitHub + Helm)

Este guia assume que você nunca usou Docker, Kubernetes, Helm ou Git antes. Vamos passar por
**tudo**: instalar as ferramentas, criar conta e repositório no GitHub, subir o código, e por
fim ver a aplicação rodando no seu computador via Kubernetes.

Sempre que aparecer `SEU_USUARIO`, troque pelo seu usuário do GitHub.
Sempre que aparecer `SEU_REPO`, troque pelo nome que você der ao repositório (ex: `k8s-helm-demo`).

---

## O que vamos construir

```
Seu código (GitHub)  --push-->  GitHub Actions builda a imagem Docker
                                        |
                                        v
                          Imagem publicada no GHCR (registro do GitHub)
                                        |
                                        v
                    Você instala com "helm install" no Kubernetes
                       que roda dentro do seu Docker Desktop
                                        |
                                        v
                         App acessível em localhost no navegador
```

---

## Parte 1 — Instalando as ferramentas

### 1.1 Git
Serve para versionar e enviar código ao GitHub.
- Windows: baixe em https://git-scm.com/downloads e instale com as opções padrão.
- Mac: abra o Terminal e rode `git --version` — se não tiver, ele oferece instalar.
- Linux: `sudo apt install git` (Ubuntu/Debian) ou equivalente da sua distro.

Verifique:
```bash
git --version
```
Configure seu nome e email (usado nos commits):
```bash
git config --global user.name "Seu Nome"
git config --global user.email "seu-email@exemplo.com"
```

### 1.2 Conta no GitHub
Se ainda não tem, crie em https://github.com/join (gratuito).

### 1.3 Docker Desktop
Baixe em https://www.docker.com/products/docker-desktop/ e instale para seu sistema operacional.

Depois de instalado, **abra o Docker Desktop** e habilite o Kubernetes:
1. Clique no ícone de engrenagem (⚙️ Settings).
2. Vá em **Kubernetes** no menu lateral.
3. Marque **Enable Kubernetes**.
4. Clique em **Apply & Restart** (pode demorar alguns minutos na primeira vez).

Você saberá que funcionou quando o ícone do Kubernetes ficar verde no canto inferior esquerdo
do Docker Desktop.

### 1.4 kubectl (já vem incluso no Docker Desktop)
É a ferramenta de linha de comando para conversar com o Kubernetes. Verifique:
```bash
kubectl version --client
kubectl config current-context
```
O segundo comando deve mostrar `docker-desktop`. Se mostrar outra coisa, rode:
```bash
kubectl config use-context docker-desktop
```

### 1.5 Helm
É o "gerenciador de pacotes" do Kubernetes — o que vamos usar para instalar a aplicação.
- Windows: `winget install Helm.Helm` (ou baixe em https://helm.sh/docs/intro/install/)
- Mac: `brew install helm`
- Linux: `sudo snap install helm --classic` (ou veja https://helm.sh/docs/intro/install/)

Verifique:
```bash
helm version
```

**Checklist antes de continuar:** `git --version`, `docker --version`, `kubectl version --client`
e `helm version` devem rodar sem erro, e `kubectl config current-context` deve mostrar `docker-desktop`.

---

## Parte 2 — Criando o repositório no GitHub

1. Acesse https://github.com/new
2. Dê um nome, por exemplo `k8s-helm-demo`.
3. Deixe como **Public** (necessário para o registro de imagem ficar público facilmente).
4. **Não** marque para criar README/gitignore automaticamente (vamos subir os nossos).
5. Clique em **Create repository**. O GitHub vai mostrar comandos — não precisa usá-los ainda,
   vamos usar os comandos abaixo.

---

## Parte 3 — Colocando o projeto no seu computador

1. Extraia o arquivo `k8s-helm-demo.zip` que te enviei em uma pasta no seu computador.
2. Abra um terminal **dentro dessa pasta** (`cd caminho/para/k8s-helm-demo`).

---

## Parte 4 — Enviando o código para o GitHub

Dentro da pasta do projeto, rode:
```bash
git init
git add .
git commit -m "primeiro commit: app, dockerfile, helm chart e workflows"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/SEU_REPO.git
git push -u origin main
```

Se pedir login, o GitHub pode abrir uma janela no navegador para autenticar, ou pedir um
**Personal Access Token** no lugar da senha (senhas comuns não funcionam mais para git push).
Para criar um token: https://github.com/settings/tokens → **Generate new token (classic)** →
marque o escopo `repo` → gere e copie o token → use-o como senha quando o git pedir.

---

## Parte 5 — Acompanhando o build automático da imagem

1. No GitHub, abra seu repositório e clique na aba **Actions**.
2. Você deve ver o workflow **"Build and Push Docker Image"** rodando (ou já concluído com
   um ✅ verde). Ele builda a imagem Docker da pasta `app/` e publica no GHCR (registro de
   containers do GitHub).
3. Isso leva 1-2 minutos.

### Tornando a imagem pública
Por padrão, o pacote gerado fica privado. Para simplificar (sem precisar configurar senha no
Kubernetes):
1. No GitHub, clique na sua foto de perfil (canto superior direito) → **Your profile**.
2. Clique na aba **Packages**.
3. Abra o pacote com o nome do seu repositório.
4. No menu **Package settings** (lateral direita), role até **Danger Zone** → **Change visibility**
   → selecione **Public** → confirme digitando o nome do pacote.

---

## Parte 6 — Ajustando o Helm chart com o seu usuário

Abra o arquivo `helm-chart/hello-app/values.yaml` e troque:
```yaml
image:
  repository: ghcr.io/SEU_USUARIO/SEU_REPO
```
pelo seu usuário e nome do repositório reais. Salve o arquivo.

Envie essa alteração para o GitHub:
```bash
git add helm-chart/hello-app/values.yaml
git commit -m "ajusta repositorio da imagem no values.yaml"
git push
```

---

## Parte 7 — Instalando a aplicação no Kubernetes local

Com o Docker Desktop aberto e o Kubernetes habilitado (Parte 1.3), rode dentro da pasta do
projeto:
```bash
helm install hello-app ./helm-chart/hello-app
```

Você deve ver uma saída parecida com:
```
NAME: hello-app
STATUS: deployed
```

### Verificando se está tudo certo
```bash
kubectl get pods
```
Espere até o status ficar `Running` (pode levar alguns segundos enquanto a imagem é baixada
do GHCR na primeira vez).

```bash
kubectl get svc
```
Você verá um serviço chamado `hello-app-hello-app`.

### Acessando a aplicação no navegador
```bash
kubectl port-forward svc/hello-app-hello-app 8080:80
```
Deixe esse comando rodando no terminal e abra no navegador:
```
http://localhost:8080
```
Você deve ver um JSON parecido com:
```json
{"message": "Hello from Kubernetes via Helm!", "hostname": "...", "timestamp": "..."}
```

🎉 Se você viu isso, parabéns — sua aplicação está rodando no Kubernetes, com a imagem vinda
do GitHub, instalada via Helm!

Para parar o port-forward, pressione `Ctrl+C` no terminal.

---

## Parte 8 — Fazendo alterações e atualizando

1. Edite algo em `app/server.js` (por exemplo, mude o texto da mensagem).
2. Envie para o GitHub:
```bash
git add app/server.js
git commit -m "atualiza mensagem da app"
git push
```
3. Acompanhe a aba **Actions** até o build da imagem terminar.
4. Atualize o deployment no cluster:
```bash
kubectl rollout restart deployment hello-app-hello-app
```
5. Repita o `kubectl port-forward` e recarregue o navegador para ver a mudança.

---

## Parte 9 — Desinstalando (quando quiser limpar)
```bash
helm uninstall hello-app
```
Isso remove tudo que foi criado no cluster (Deployment, Service, Pods).

---

## Solução de problemas comuns

**`ImagePullBackOff` ou `ErrImagePull` no `kubectl get pods`**
→ Geralmente significa que a imagem ainda é privada. Volte na Parte 5 e torne o pacote público,
ou configure o `imagePullSecrets` (veja seção "Imagem privada" no README principal).

**`kubectl` não conecta / erro de conexão**
→ Confirme que o Docker Desktop está aberto e o Kubernetes está com o ícone verde. Rode
`kubectl config use-context docker-desktop`.

**`helm install` diz que já existe um release com esse nome**
→ Rode `helm uninstall hello-app` antes de instalar de novo, ou escolha outro nome:
`helm install outro-nome ./helm-chart/hello-app`.

**Git pede usuário/senha e a senha não funciona**
→ O GitHub não aceita mais senha normal via linha de comando. Use um Personal Access Token
(veja Parte 4) ou configure autenticação via SSH.

---

## Glossário rápido

- **Docker**: empacota sua aplicação e tudo que ela precisa (dependências, runtime) em uma
  "imagem", que roda de forma idêntica em qualquer máquina.
- **Kubernetes (k8s)**: orquestra containers — decide onde rodar, reinicia se cair, escala
  réplicas, etc. O Docker Desktop roda uma versão local dele no seu computador.
- **Helm**: um "instalador" para Kubernetes. Em vez de escrever vários arquivos YAML na mão,
  você usa um "chart" (modelo reutilizável) com valores configuráveis.
- **GHCR (GitHub Container Registry)**: onde as imagens Docker ficam guardadas, dentro do
  próprio GitHub.
- **GitHub Actions**: automação que roda em resposta a eventos (como um `git push`) — no nosso
  caso, builda e publica a imagem automaticamente.
- **Deployment**: recurso do Kubernetes que garante que N cópias ("réplicas") do seu container
  estejam rodando.
- **Service**: recurso do Kubernetes que dá um endereço de rede estável para acessar os pods.
- **Pod**: a menor unidade do Kubernetes — normalmente um container rodando.
