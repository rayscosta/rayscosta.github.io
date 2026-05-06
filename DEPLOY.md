# Guia de Deploy — AWS Amplify + S3

Este guia cobre três formas de servir este site estático na AWS.

---

## Opção 1 — AWS Amplify (recomendada para CI/CD)

O Amplify conecta diretamente ao GitHub e faz deploy automático a cada push.

### Passo a passo

1. Acesse o console da AWS → **Amplify**
2. Clique em **"New app" → "Host web app"**
3. Escolha **GitHub** como fonte e autorize o acesso
4. Selecione o repositório `rayscosta/rayscosta.github.io` e a branch `main`
5. Na tela de configuração de build, o Amplify detecta o `amplify.yml` automaticamente
6. Clique em **"Save and deploy"**

> O Amplify gera uma URL pública do tipo `https://main.xxxxxxx.amplifyapp.com`.
> Você pode configurar um domínio customizado em **Domain management**.

### Deploy automático

Após a configuração inicial, cada `git push` para a branch `main` dispara um novo deploy. Ideal para laboratório e testes contínuos.

---

## Opção 2 — Amazon S3 Static Website Hosting

Útil para entender fundamentos de armazenamento e políticas IAM.

```bash
# 1. Configure o AWS CLI
aws configure
# Informe: Access Key ID, Secret Access Key, região (ex: us-east-1), formato: json

# 2. Crie o bucket (nome único globalmente)
aws s3 mb s3://raycosta-portfolio --region us-east-1

# 3. Habilite o hosting estático
aws s3 website s3://raycosta-portfolio \
  --index-document index.html \
  --error-document index.html

# 4. Desabilite o Block Public Access
aws s3api put-public-access-block \
  --bucket raycosta-portfolio \
  --public-access-block-configuration \
    "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# 5. Bucket policy de leitura pública
aws s3api put-bucket-policy --bucket raycosta-portfolio --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::raycosta-portfolio/*"
  }]
}'

# 6. Upload dos arquivos
aws s3 sync . s3://raycosta-portfolio \
  --exclude ".git/*" --exclude "*.md" --exclude "amplify.yml"
```

URL: `http://raycosta-portfolio.s3-website-us-east-1.amazonaws.com`

> Para HTTPS no S3, coloque o **CloudFront** na frente do bucket.

---

## Opção 3 — EC2 com Apache2 (já existente)

```bash
# Copiar arquivos via SCP
scp -i ~/sua-chave.pem \
  index.html style.css contato.html foto_perfil.jpg \
  logo-linkedin.png logo-github.png logo-instagram.png \
  ubuntu@<IP-DA-EC2>:/var/www/html/

# Ou via rsync (mais eficiente para updates)
rsync -avz --exclude='.git' --exclude='*.md' \
  -e "ssh -i ~/sua-chave.pem" \
  ./ ubuntu@<IP-DA-EC2>:/var/www/html/
```

```bash
# Na EC2
sudo systemctl status apache2
sudo systemctl restart apache2
# Acesse: http://<IP-PÚBLICO-DA-EC2>
```

> Para HTTPS na EC2 com Certbot (requer domínio):
> ```bash
> sudo apt install certbot python3-certbot-apache -y
> sudo certbot --apache -d seudominio.com
> ```

---

## Comparativo

| | Amplify | S3 | EC2 + Apache |
|---|---|---|---|
| CI/CD automático | ✅ | ❌ (manual) | ❌ (manual) |
| HTTPS nativo | ✅ | ⚠️ CloudFront | ⚠️ Certbot |
| Custo | Free tier generoso | Muito baixo | EC2 já pago |
| Complexidade | Baixa | Média | Média |
| Melhor para aprender | Deploy moderno | Storage + IAM | Infra clássica |

---

## Atualizando o site

### Via Amplify (automático)
```bash
git push origin main
```

### Via S3 (manual)
```bash
aws s3 sync . s3://raycosta-portfolio --exclude ".git/*" --exclude "*.md"
```

### Via EC2 (manual)
```bash
rsync -avz --exclude='.git' -e "ssh -i ~/sua-chave.pem" \
  ./ ubuntu@<IP>:/var/www/html/
```
