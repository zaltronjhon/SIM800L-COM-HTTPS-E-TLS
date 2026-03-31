# SIM800L-COM-HTTPS-E-TLS
Senhores, bom dia!
 Após a compra de um arduino Ttgo T-call V1.3 Esp32 com uma Sim800l em pleno 2026, notei que a criptografia não era mais suportada pelo modulo Sim800L, modulo esse o qual pode ser colocado a parte em uma placa convecional de arduino, apesar de ser um modulo 2G, minha cidade tem uma grande cobertura de sinal.


 Primeiramente, quero informar que é sim possível fazer "TLS  1.2 usando SIM800L" e outros tipos de certificados, no entanto, o SIM800L vai servir apenas como um cano de dados, e não tera mais a responsabilidade de fazer o TLS, quem fara toda parte inteligênte sera a sua ESP32. que é o meu caso. Assim conseguimos dar longevidade a vida desse chip.

 Vou por abaixo as bibliotecas que utilizei, junto com um resumo feito pelo GPT, e se os senhores quiserem é só pegar a logica e jogar em alguma IA que ela fara todo trabalho para os senhores.

 Detalhes do meu projeto:
 A proposta era criar um projeto IOT com sensores na minha cidade, com isso os dados coletados seriam enviados para meu endpoint, que era um googlesheets provisoriamente com um script inserido nele, assim a minha primeira versão eu mandava os dados compactados dentro de um JSON, funcionava, e funciona, mas como os dados que eu precisava eram poucos, eu comecei a mandar via URL, assim toda vez que tivesse o sinal, ele mandava via url e o dado saia tabulado na planilha.

 FIM


1. Configuração do Hardware e Bibliotecas

•
Hardware: Certifique-se de estar utilizando uma placa ESP32, como a TTGO T-Call v1.3, que integra o módulo SIM800L.

•
Bibliotecas: Instale as seguintes bibliotecas no seu ambiente de desenvolvimento Arduino IDE ou PlatformIO:

•
TinyGsmClient: Para gerenciar a conexão GPRS do SIM800L.

•
ESP_SSLClient: Para que o ESP32 realize a criptografia SSL/TLS.

•
ArduinoHttpClient: Para facilitar o envio de requisições HTTP/HTTPS e cabeçalho

---------CODIGO
```
#include <TinyGsmClient.h>
#include <ESP_SSLClient.h>
#include <ArduinoHttpClient.h>
#include "config.h" // Importa suas chaves secretas localmente

// Pinos TTGO T-Call v1.3 (Ajuste conforme sua versão)
#define MODEM_TX             27
#define MODEM_RX             26
#define MODEM_POWER_ON       23

HardwareSerial modemSerial(1);
TinyGsm modem(modemSerial);
TinyGsmClient client(modem);
ESP_SSLClient secure_layer;

void setup() {
  Serial.begin(115200);
  modemSerial.begin(115200, SERIAL_8N1, MODEM_RX, MODEM_TX);

  // Liga o modem (Power ON)
  pinMode(MODEM_POWER_ON, OUTPUT);
  digitalWrite(MODEM_POWER_ON, HIGH);
  
  Serial.println("Conectando GPRS...");
  if (!modem.gprsConnect(MY_APN, MY_USER, MY_PASS)) {
    Serial.println("Falha na conexão GPRS");
    return;
  }

  // Configuração de Segurança
  secure_layer.setInsecure(); // Ignora validação de certificado para poupar RAM
  secure_layer.setClient(&client);
}

void loop() {
  if (modem.isGprsConnected()) {
    Serial.println("Conectando ao Supabase via SSL...");
    
    if (secure_layer.connect(SUPABASE_HOST, 443)) {
      HttpClient http = HttpClient(secure_layer, SUPABASE_HOST, 443);

      // JSON de exemplo com os dados do sensor
      int sensorValue = analogRead(32); 
      String postData = "{\"nivel\": " + String(sensorValue) + ", \"device\": \"" + String(DEVICE_ID) + "\"}";
      
      Serial.println("Enviando POST...");
      
      http.beginRequest();
      http.post(SUPABASE_PATH);
      
      // CABEÇALHOS DE SEGURANÇA (Obrigatórios para Supabase)
      http.sendHeader("Host", SUPABASE_HOST);
      http.sendHeader("Content-Type", "application/json");
      http.sendHeader("apikey", SUPABASE_KEY);
      http.sendHeader("Authorization", "Bearer " + String(SUPABASE_KEY));
      http.sendHeader("Content-Length", postData.length());
      
      http.endRequest();
      http.print(postData);

      // Resposta do Servidor
      int statusCode = http.responseStatusCode();
      String response = http.responseBody();

      Serial.printf("Status: %d | Resposta: %s\n", statusCode, response.c_str());
      
      http.stop();
    } else {
      Serial.println("Falha na conexão SSL com o Host.");
    }
  }

  // Espera 1 hora (ou use Deep Sleep como no código anterior)
  delay(3600000); 
}
```


Pontos Chave no Código:

•
secure_layer.setInsecure();: Essencial para ignorar a validação de certificados e permitir a conexão com servidores modernos.

•
secure_layer.connect(client, "your_supabase_host", 443): Estabelece a conexão SSL/TLS usando o ESP32. Substitua your_supabase_host pelo host real do seu projeto Supabase.

•
HttpClient http = HttpClient(secure_layer, "your_supabase_host", 443 );: Inicializa o cliente HTTP com a camada de segurança SSL fornecida pelo ESP_SSLClient.

•
http.sendHeader("Host", "your_supabase_host" );: Crucial para o SNI, garantindo que a Cloudflare direcione a requisição corretamente. Substitua your_supabase_host pelo host real.

•
http.post("/rest/v1/your_edge_function_path", contentType, postData );: Envia a requisição POST. Adapte o caminho (/rest/v1/your_edge_function_path) e o postData conforme a sua Edge Function ou endpoint do Supabase.

3. Considerações para Outros Servidores HTTPS

Se você pretende conectar seu ESP32 a outros serviços HTTPS (Google Sheets, AWS, outros bancos de dados), considere os seguintes pontos:

•
Identificar Host e Resource: Para uma URL como https://api.exemplo.com/v1/data, o Host é api.exemplo.com e o Resource é /v1/data. Adapte-os no seu código.

•
Manter setInsecure( ): A menos que você tenha um motivo específico e recursos para gerenciar certificados, mantenha secure_layer.setInsecure().

•
Ajustar Headers: Sempre verifique se o servidor exige cabeçalhos específicos (como Host, Content-Type, Authorization, apikey).

•
Definir JSON de Envio: Prepare a string postData com o formato JSON esperado pelo novo servidor.

•
Verificar Autenticação: Se o serviço exigir autenticação (API Key, Bearer Token), adicione o cabeçalho Authorization ou apikey conforme a documentação do serviço.




TinyGsmClient
Gerencia a conexão GPRS do SIM800L, fornecendo acesso à internet (TIM).

ESP_SSLClient
Realiza o trabalho pesado de criptografia (TLS 1.2) no ESP32.

setInsecure()
Ignora a necessidade de validação de certificados digitais, otimizando memória.

setBufferSizes
(Implícito) Reserva memória no ESP32 para evitar travamentos durante HTTPS.

ArduinoHttpClient
Facilita a construção e envio de requisições HTTP/HTTPS com cabeçalhos.


Conclusão

Ao delegar a inteligência da criptografia SSL/TLS para o ESP32 e gerenciar corretamente os cabeçalhos de requisição, é possível estabelecer uma comunicação HTTPS robusta e eficiente com serviços modernos, mesmo utilizando hardware como o SIM800L. Este método garante a segurança dos dados e a compatibilidade com infraestruturas.

