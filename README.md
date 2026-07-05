# Demo: Deploy no Kubernetes do Docker Desktop via Helm + GitHub

Este projeto mostra o fluxo completo:
1. App simples em Node.js containerizada.
2. GitHub Actions builda a imagem e publica no GitHub Container Registry (GHCR).
3. Um Helm chart descreve como rodar essa imagem no cluster.
4. Você faz `helm install` apontando para a imagem publicada, rodando no k8s local do Docker Desktop.

## Passo 1 — Habilitar Kubernetes no Docker Desktop
No Docker Desktop: Settings → Kubernetes → marque "Enable Kubernetes" → Apply & Restart.
Confirme com:
```bash
kubectl config current-context   # deve mostrar docker-desktop
kubectl get nodes
```

## Passo 2 — Criar o repositório no GitHub
1. Crie um repositório novo (ex: `k8s-helm-demo`) no GitHub.
2. Copie todos os arquivos deste projeto para dentro dele.
3. Faça commit e push para a branch `main`:
```bash
git init
git add .
git commit -m "app inicial + chart helm + workflow"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/SEU_REPO.git
git push -u origin main
```

## Passo 3 — GitHub Actions builda e publica a imagem
O workflow em `.github/workflows/docker-build-push.yml` já está configurado para:
- Disparar a cada push na branch `main` que mexa em `app/**`.
- Buildar a imagem Docker a partir de `app/Dockerfile`.
- Publicar em `ghcr.io/SEU_USUARIO/SEU_REPO:latest`.

Ele usa o `GITHUB_TOKEN` automático do repositório — não precisa configurar nenhum secret manualmente.

Depois do primeiro push, acompanhe em: aba **Actions** do seu repositório no GitHub.

### Tornando o pacote público (recomendado para simplificar)
Por padrão, pacotes no GHCR ficam privados. Para não precisar configurar `imagePullSecrets` no cluster:
1. No GitHub, vá em **seu perfil → Packages**.
2. Abra o pacote criado (nome do seu repositório).
3. Em **Package settings**, mude a visibilidade para **Public**.

Se preferir manter privado, veja a seção "Imagem privada" abaixo.

## Passo 4 — Ajustar o Helm chart
Edite `helm-chart/hello-app/values.yaml` e troque:
```yaml
image:
  repository: ghcr.io/SEU_USUARIO/SEU_REPO
```
pelo caminho real da sua imagem publicada.

## Passo 5 — Instalar com Helm no cluster local
Com o Docker Desktop Kubernetes ativo:
```bash
helm install hello-app ./helm-chart/hello-app
```

Verifique:
```bash
kubectl get pods
kubectl get svc
```

Para acessar a app (Service tipo NodePort):
```bash
kubectl port-forward svc/hello-app-hello-app 8080:80
```
Depois abra `http://localhost:8080` no navegador.

## Passo 6 — Atualizando depois de mudar o código
1. Faça alterações em `app/`.
2. Commit e push — o Actions builda e publica uma nova tag da imagem automaticamente.
3. Atualize o release no cluster:
```bash
helm upgrade hello-app ./helm-chart/hello-app
```
Se quiser forçar novo pull de `latest`:
```bash
kubectl rollout restart deployment hello-app-hello-app
```

## (Opcional) Imagem privada no GHCR
Se preferir manter a imagem privada, crie um Personal Access Token com escopo `read:packages` e gere o secret no cluster:
```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=SEU_USUARIO \
  --docker-password=SEU_TOKEN \
  --docker-email=seu-email@exemplo.com
```
E em `values.yaml`:
```yaml
imagePullSecrets:
  - name: ghcr-secret
```

## Publicando o Helm chart no GitHub (repositório Helm via GitHub Pages)

Isso permite instalar o chart de qualquer lugar com:
```bash
helm repo add hello-app https://SEU_USUARIO.github.io/SEU_REPO/
helm install hello-app hello-app/hello-app
```

### Passo A — Criar a branch `gh-pages`
Ela vai hospedar o índice do repositório Helm (arquivo `index.yaml` + os `.tgz` dos charts).
```bash
git checkout --orphan gh-pages
git rm -rf .
git commit --allow-empty -m "inicializa gh-pages"
git push origin gh-pages
git checkout main
```

### Passo B — Habilitar o GitHub Pages
No repositório: **Settings → Pages → Build and deployment → Source: "Deploy from a branch"**, selecione a branch `gh-pages` e pasta `/ (root)`.
Sua URL de Pages ficará algo como:
```
https://SEU_USUARIO.github.io/SEU_REPO/
```

### Passo C — O workflow `release-chart.yml` faz o resto
Já incluído neste projeto (`.github/workflows/release-chart.yml`), ele usa a Action oficial `chart-releaser-action`:
- Dispara a cada push em `main` que altere algo em `helm-chart/**`.
- Empacota o chart (`helm package`) em um `.tgz`.
- Cria uma **GitHub Release** com esse `.tgz` anexado.
- Atualiza o `index.yaml` na branch `gh-pages`, apontando pro `.tgz` da release.

Não precisa configurar nenhum secret — ele usa o `GITHUB_TOKEN` padrão do repositório (que já tem permissão de escrita habilitada no workflow).

**Importante:** antes do primeiro push, garanta que a versão em `helm-chart/hello-app/Chart.yaml` (`version: 0.1.0`) seja incrementada a cada mudança que você quiser publicar — o chart-releaser só cria uma nova release se a versão mudar.

### Passo D — Usar o repositório publicado
Depois que a Action rodar com sucesso (acompanhe na aba **Actions**):
```bash
helm repo add hello-app https://SEU_USUARIO.github.io/SEU_REPO/
helm repo update
helm search repo hello-app
helm install hello-app hello-app/hello-app
```

Para atualizar depois de uma nova versão do chart:
```bash
helm repo update
helm upgrade hello-app hello-app/hello-app
```

## Alternativa mais simples: OCI registry no GHCR
Em vez de GitHub Pages, dá pra publicar o chart direto como artefato OCI no GitHub Container Registry — sem precisar de branch `gh-pages` nem Pages habilitado:
```bash
helm package helm-chart/hello-app
helm push hello-app-0.1.0.tgz oci://ghcr.io/SEU_USUARIO
```
E instalar com:
```bash
helm install hello-app oci://ghcr.io/SEU_USUARIO/hello-app --version 0.1.0
```
Isso também pode ser automatizado via Actions, se preferir esse caminho em vez do GitHub Pages — me avise que eu monto o workflow.
