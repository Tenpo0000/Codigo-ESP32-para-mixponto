# MixPonto — Terminal RFID com ESP32

Firmware para um terminal de ponto com **ESP32 clássico**, leitor **RC522**, display **LCD I2C 16×2**, LED e buzzer. O funcionário aproxima o cartão, o terminal consulta o vínculo no Firebase e envia o registro ao Realtime Database.

O código desta pasta está em [`PontoESP32.ino`](PontoESP32.ino). Ele autentica com a conta titular e não utiliza o login individual do funcionário.

> **Compatibilidade:** este arquivo usa o modelo antigo `contas/{UID_DO_TITULAR}`. Ele não resolve o ID numérico da conta nem contas com chave interna independente do UID. Também não implementa o controle de intervalo compartilhado com o aplicativo Android. Confira essas limitações antes de usar este firmware com a versão atual do sistema.

## Funcionamento

1. Conecta ao Wi-Fi, sincroniza o relógio por NTP e autentica no Firebase.
2. Lê o UID do cartão e o normaliza em hexadecimal maiúsculo, preservando zeros à esquerda.
3. Consulta o funcionário vinculado ao cartão na conta do titular.
4. Confere se o vínculo também existe no cadastro do funcionário e se ele está ativo.
5. Prepara uma chave de evento e salva uma única batida pendente na memória persistente do ESP32.
6. Envia o registro ao Firebase e só limpa a pendência após confirmação de sucesso.
7. Exibe o resultado no LCD e emite sinal sonoro e luminoso.

Enquanto existe uma batida pendente, o terminal bloqueia novas leituras e tenta reenviar a mesma chave a cada 15 segundos. O bloqueio local de repetição do mesmo cartão dura 10 segundos; isso não equivale ao intervalo compartilhado de cinco minutos do sistema mobile.

## Hardware e ligações

### RC522 — SPI

| Pino do RC522 | ESP32 |
| --- | --- |
| SDA / SS | GPIO 5 |
| SCK | GPIO 18 |
| MOSI | GPIO 23 |
| MISO | GPIO 19 |
| RST | GPIO 4 |
| GND | GND comum |
| Alimentação | 3,3 V, conforme o módulo utilizado |
| IRQ | Não utilizado |

O pino chamado **SDA** no RC522 exerce a função **SS** nesta montagem SPI; não deve ser confundido com o SDA do LCD I2C.

O MFRC522 trabalha com alimentação típica de 3,3 V. Não alimente o módulo RC522 diretamente com 5 V sem uma especificação explícita do fabricante do módulo. Consulte o [datasheet da NXP](https://www.nxp.com/docs/en/data-sheet/MFRC522.pdf).

### LCD e sinalização

| Componente | Ligação/configuração |
| --- | --- |
| LCD SDA | GPIO 21 |
| LCD SCL | GPIO 22 |
| LCD | I2C, endereço `0x27`, 16 colunas e 2 linhas |
| LED | GPIO 27, com resistor adequado |
| Buzzer | Sinal no GPIO 26; o código gera tom de 5 kHz |

Todos os módulos devem compartilhar GND. Dimensione alimentação e acionamento do buzzer conforme seus componentes; não presuma que qualquer carga pode ser alimentada diretamente pelo GPIO.

Confira a tensão dos pull-ups do módulo I2C do LCD: sinais ligados ao ESP32 precisam respeitar seus limites elétricos. Um backpack alimentado em 5 V pode exigir conversão de nível. Consulte a [documentação elétrica do ESP32](https://documentation.espressif.com/esp32_datasheet_en.html) antes da montagem.

## Bibliotecas e ambiente

O sketch usa Arduino para ESP32 e inclui:

| Biblioteca/cabeçalho | Uso |
| --- | --- |
| `Wire.h` | Barramento I2C do LCD |
| `SPI.h` | Comunicação com o RC522 |
| `WiFi.h` | Conexão de rede |
| `Preferences.h` | Pendência persistida na flash/NVS |
| `LiquidCrystal_I2C.h` | Controle do LCD |
| `MFRC522.h` | Leitura RFID |
| `FirebaseESP32.h` | Autenticação e operações no Realtime Database |
| `esp_system.h`, `time.h` | Identificador do evento e relógio |

Instale o suporte **esp32 by Espressif Systems** no gerenciador de placas da Arduino IDE, conforme a [documentação oficial](https://docs.espressif.com/projects/arduino-esp32/en/latest/installing.html). Selecione a placa correspondente ao seu ESP32 clássico; não presuma a mesma pinagem para ESP32-C3/S3.

Instale também as bibliotecas externas:

- [Firebase ESP32 Client, de Mobizt](https://github.com/mobizt/Firebase-ESP32), que fornece `FirebaseESP32.h`.
- [MFRC522](https://github.com/miguelbalboa/rfid).
- Uma implementação de `LiquidCrystal_I2C` compatível com o construtor e as chamadas `init()` e `backlight()` presentes no sketch.

**Atenção à biblioteca Firebase:** o projeto do Mobizt utilizado aqui está marcado como descontinuado pelo autor. Este sketch não usa `Firebase_ESP_Client.h` nem a API de outras bibliotecas Firebase mais novas. Substituí-la exige adaptação do código. Não há versões de dependências fixadas ou compilação reproduzível documentada nesta pasta.

## Configuração

Copie o exemplo sem segredos para a configuração local:

```powershell
Copy-Item config.example.h config.h
```

Preencha os valores em `config.h`. Esse arquivo é incluído pelo sketch e ignorado pelo Git; `config.example.h` pode ser publicado.

| Definição | Conteúdo esperado |
| --- | --- |
| `WIFI_SSID` | Nome da rede Wi-Fi |
| `WIFI_PASSWORD` | Senha da rede |
| `USER_EMAIL` | E-mail da identidade titular usada pelo terminal |
| `USER_PASSWORD` | Senha dessa identidade |
| `API_KEY` | Configuração de API do projeto Firebase |
| `DATABASE_URL` | URL da instância Realtime Database |
| `EMPRESA_UID` | UID da identidade titular, também usado como chave da conta nesta versão |

O UID autenticado precisa coincidir com `EMPRESA_UID`. Essa conferência impede continuar com outro login, mas não substitui as regras do banco.

Não utilize no firmware uma chave JSON de conta de serviço, Firebase Admin SDK ou segredo administrativo do banco. O código usa autenticação por e-mail e senha.

## Preparação no sistema web

Para o modelo de dados suportado por este arquivo:

```text
contas/
  {UID_DO_TITULAR}/
    cartoes/
      {UID_DO_CARTAO}: "{funcionarioId}"
    funcionarios/
      {funcionarioId}/
        nome: "Funcionário de exemplo"
        status: "ativo"
        cartaoUid: "{UID_DO_CARTAO}"
```

Gerencie os vínculos pelo cadastro de funcionários do site. O firmware exige coerência entre o índice `cartoes` e o campo `cartaoUid`; um índice antigo isolado não basta para registrar ponto.

As regras publicadas precisam autorizar a identidade titular nos caminhos necessários. Não torne o banco público para contornar erros. Este repositório não inclui um arquivo de regras Firebase.

## Compilar e gravar

1. Abra `PontoESP32.ino` na Arduino IDE.
2. Se a IDE solicitar uma pasta de sketch com o mesmo nome, crie/use `PontoESP32/PontoESP32.ino` para a cópia de trabalho e coloque `config.h` na mesma pasta do `.ino`.
3. Instale as dependências e selecione a placa e a porta USB corretas.
4. Confira as ligações e preencha a configuração local em `config.h`.
5. Use **Verificar** para compilar; depois, **Carregar** para gravar no dispositivo.
6. Abra o monitor serial em **115200 baud**.
7. Aguarde a mensagem **SISTEMA DE PONTO / APROXIME CARTAO** e teste um cartão vinculado em um ambiente de validação.

Antes de alterar firmware ou apagar a flash, confira se existe uma batida pendente. Apagar a NVS pode eliminar a única cópia local de um registro ainda não confirmado.

## Registros e horário

O destino das batidas é:

```text
contas/{UID_DO_TITULAR}/registros/{AAAA-MM-DD}/{funcionarioId}/{evento}
```

Exemplo fictício do conteúdo:

```json
{
  "nome": "Funcionário de exemplo",
  "timestamp": "2026-10-21T12:00:00.000Z",
  "uidCartao": "A1B2C3D4",
  "origem": "esp32"
}
```

A data do caminho usa o horário configurado como `BRT3` (UTC−3). O timestamp é gerado pelo relógio do ESP32 e enviado em UTC com sufixo `Z`. Os servidores NTP configurados são `pool.ntp.org` e `time.google.com`.

O horário desta versão **não é um timestamp atribuído pelo Firebase no momento da gravação**. A verificação local detecta um relógio não inicializado, mas não garante sua precisão absoluta.

A pendência é armazenada com `Preferences`, no namespace `mixponto`, chave `pendente`. O reenvio usa `setJSON` no mesmo caminho para evitar criar outro evento pela simples repetição da transmissão.

## Mensagens comuns

| Mensagem | Verificação |
| --- | --- |
| `SEM WIFI` | Rede configurada, alcance e alimentação |
| `SINCRONIZANDO` / `SEM HORARIO NTP` | Acesso à rede e aos servidores de horário |
| `AUTENTICANDO` | Credenciais, configuração Firebase e conectividade |
| `CONTA DIFERENTE` | UID autenticado, `EMPRESA_UID` e conta da pendência |
| `CARTAO SEM LINK` | Índice do cartão e vínculo no funcionário |
| `FUNC. INATIVO` | Situação do funcionário no site |
| `PONTO PENDENTE` | Rede e permissão de gravação; outros cartões ficam bloqueados |
| `ERRO MEMORIA` | Persistência local; não apagar a flash antes de conferir a pendência |
| `PONTO REGISTRADO` | Gravação confirmada pelo Firebase e pendência local removida |

O monitor serial mostra o UID do cartão para auxiliar no vínculo. Evite divulgar logs com identificadores ou dados de funcionários.

## Limitações e evolução

- O código identifica a conta pelo UID do titular. Contas novas com chave interna independente ou ID numérico exigem atualização do firmware antes do uso.
- Não existe reserva/transação em `controleBatidas` ou verificação do intervalo compartilhado com Android. O bloqueio de 10 segundos atua somente no último cartão lido por este terminal.
- O terminal valida funcionário ativo, mas não consulta a admissão para decidir se aceita uma batida.
- Não aceita cartões novos sem rede: precisa consultar o vínculo no Firebase. A pendência cobre a recuperação de uma transmissão, não uma fila offline de vários funcionários.
- Uma pendência de outra conta bloqueia o envio; não a descarte sem conferir o registro e o vínculo original.
- O UID RFID é usado como identificador; o sketch não implementa autenticação criptográfica do cartão ou biometria.
- Não há sincronização automática com futuras mudanças de regras, permissões ou identificadores do site.
- Esta documentação foi conferida contra o código da pasta. Não foi realizada compilação nem teste físico como parte da criação deste README.

## Publicação e licenças

As credenciais foram separadas do sketch: `config.h` é privado e ignorado pelo Git, enquanto `config.example.h` contém campos vazios para configuração. Não envie `config.h` nem cópias dele ao GitHub. Ignorar esse arquivo não remove segredos de commits antigos ou de arquivos já compartilhados.

As bibliotecas mantêm suas próprias licenças. Esta pasta não contém uma licença geral do projeto; não se presume que a licença de uma dependência se aplique ao firmware inteiro.
