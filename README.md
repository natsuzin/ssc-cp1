# SSC — Case Prático 1: Avaliação de Vulnerabilidades em uma Aplicação Web

Ambiente de testes para a avaliação de segurança do **OWASP Juice Shop**, realizada na disciplina de Segurança de Sistemas Computacionais (UNIVALI).

> ⚠️ **Escopo:** os testes descritos aqui devem ser executados **apenas** contra a instância local do Juice Shop, implantada no contêiner isolado abaixo. Testar sistemas de terceiros sem autorização é crime.

## Pré-requisitos

- Docker Desktop instalado, com WSL2 habilitado (Windows)
- Ferramentas de segurança instaladas na máquina de teste:
  - [OWASP ZAP](https://www.zaproxy.org/)
  - [Burp Suite Community](https://portswigger.net/burp/communitydownload)
  - [sqlmap](https://github.com/sqlmapproject/sqlmap)
  - [Nikto](https://github.com/sullo/nikto) (ou a imagem Docker `frapsoft/nikto`)

## 1. Subindo o ambiente

O `docker-compose.yml` do projeto sobe o Juice Shop em uma rede Docker isolada (`pentest-net`), expondo a porta apenas em `127.0.0.1:3000`.

```yaml
services:
  juice-shop:
    image: bkimminich/juice-shop:latest
    container_name: case1-juice-shop
    ports:
      - "127.0.0.1:3000:3000"
    restart: unless-stopped
    networks:
      - pentest-net
networks:
  pentest-net:
    name: pentest-net
    driver: bridge
```

Passo a passo:

1. Clone o repositório e entre na pasta do projeto:
   ```bash
   git clone https://github.com/natsuzin/ssc-cp1.git
   cd ssc-cp1
   ```
2. Suba o ambiente:
   ```bash
   docker compose up -d
   ```
3. Verifique se o contêiner está `running`:
   ```bash
   docker compose ps
   ```
4. Acesse `http://localhost:3000` no navegador e confirme que a aplicação carrega.

Para acompanhar logs ou encerrar o ambiente:

```bash
docker compose logs -f     # acompanhar logs
docker compose down        # derrubar o ambiente
```

### Endereços de rede

| Item | Descrição |
|---|---|
| Aplicação alvo | OWASP Juice Shop (`bkimminich/juice-shop:latest`) |
| Rede Docker | `pentest-net` (bridge isolada) |
| IP máquina de teste | `127.0.0.1` (loopback) |
| Porta exposta (alvo) | `3000` — `http://localhost:3000` |
| Proxy Burp Suite | `127.0.0.1:8080` |
| Proxy OWASP ZAP | `127.0.0.1:8081` |

## 2. Rodando os scripts de análise

### OWASP ZAP — Automated Scan

Usado para o levantamento inicial de vulnerabilidades em vários endpoints (achados V-06, V-07 e V-08 do relatório).

1. Abra o ZAP e vá em **Automated Scan**.
2. Informe `http://localhost:3000` como URL alvo.
3. Clique em **Attack** e aguarde a varredura concluir.
4. Os alertas ficam disponíveis na aba lateral (ex.: CORS mal configurado, CSP ausente, divulgação de timestamp).

O ZAP também pode ser usado como proxy de interceptação (porta `8081`) para apoiar testes manuais com o navegador.

### sqlmap — confirmação de SQL Injection

Usado para automatizar e confirmar a injeção de SQL no login (achado V-01).

1. Capture uma requisição de login (`POST /rest/user/login`) via Burp Suite ou ZAP e salve como `login_request.txt`.
2. Execute:
   ```bash
   python sqlmap.py -r login_request.txt -p email --batch --dbms=SQLite --level=5 --risk=3 --ignore-code=401 --string="token"
   ```

Observações sobre os parâmetros:
- `--ignore-code=401`: evita que o sqlmap pare ao receber respostas 401, esperadas em tentativas de login malsucedidas.
- `--string="token"`: distingue respostas verdadeiras de falsas pela presença da palavra `"token"` no corpo da resposta, retornada apenas em logins bem-sucedidos.

### Nikto — varredura do servidor

Usado para identificar configurações inseguras e arquivos expostos (achados V-03 e V-06), como complemento independente ao ZAP.

Via Docker:
```bash
docker run --rm frapsoft/nikto -h http://host.docker.internal:3000
```

Ou, com o Nikto instalado localmente:
```bash
nikto -h http://localhost:3000
```

### Burp Suite

Não possui um script de linha de comando — é usado interativamente como proxy de interceptação:

1. Configure o navegador para usar o proxy `127.0.0.1:8080`.
2. Ative o **Intercept** na aba **Proxy**.
3. Navegue pela aplicação para capturar e inspecionar/manipular requisições (ex.: `GET /rest/admin/application-configuration`, `POST /rest/user/login`).

### Navegador com DevTools

Usado de forma complementar, sem script associado — inspecione requisições, cookies e o `Local Storage` (Aba **Application** no Chrome/Edge) para confirmar achados como o token JWT armazenado no cliente (V-09).

## Estrutura esperada do repositório

```
ssc-cp1/
├── docker-compose.yml
├── README.md
└── (evidências, requests salvas, saídas de scripts, etc.)
```
