# framework_app

Aplicação comum da prova de conceito com SHT31-D, SSD1306, botão, LED e
telemetria. Os mesmos fontes C são compilados para cada plataforma do framework.
O repositório é independente do nome usado no checkout: nos projetos de firmware,
ele deve ser incluído como submódulo em `src/app/`.

## Organização

| Diretório | Responsabilidade |
| --- | --- |
| `acquisition/` | Aquisição periódica, validação do resultado e preservação da última amostra válida. |
| `button/` | Detecção de acionamentos estáveis e solicitação de troca de tela. |
| `gui/` | Seleção de tela, renderização e atualização incremental do SSD1306. |
| `status_led/` | Indicação de inicialização, operação e erros I2C. |
| `telemetry/` | Fila de registros e política de repetição ou descarte na UART. |

## Dependências e fronteiras

A aplicação depende da biblioteca padrão C e das interfaces públicas dos
componentes `button`, `led`, `sht31d`, `ssd1306` e `telemetry`. Esses componentes
permanecem em repositórios separados, selecionados pelo projeto de firmware.
Não há submódulos de componentes dentro deste repositório.

Os arquivos não incluem SDK, BSP, porte da HAL ou RTOS. Pinos, barramentos,
endereços dos dispositivos, construção de recursos, arena da HAL e inicialização
da plataforma pertencem ao BSP e à raiz de composição do firmware.

## Integração

Adicione o repositório ao projeto de firmware mantendo o nome local `app`:

```sh
git submodule add git@github.com:fmsantos8/framework_app.git src/app
```

O projeto integrador fornece C11 ou posterior, a biblioteca C, os diretórios
de inclusão dos componentes e o diretório raiz deste submódulo. Também inclui
explicitamente os fontes da aplicação em seu sistema de compilação. As opções
do ESP-IDF, STM32Cube e nRF Connect SDK permanecem no projeto integrador.

`main.c` constrói os recursos do BSP, inicializa os componentes e fornece a
função de tempo aos módulos que a utilizam. Depois inicializa a aplicação,
inicia a base de tempo e executa sucessivamente:

1. `app_button_run()`;
2. `app_acquisition_run()`;
3. `app_gui_run()`;
4. `app_status_led_run()`;
5. `app_telemetry_run()`.

Os handles dos componentes são passados às respectivas chamadas. A função
de tempo retorna milissegundos nominais em `uint32_t`, com contagem módulo
2^32; sua sincronização pertence à integração. As notificações do temporizador
não garantem, por si só, uma contagem exata do tempo físico decorrido.

Cada módulo mantém uma única instância de estado estático por firmware.
Inicialização e processamento são sequenciais, fora de interrupções. No Zephyr,
uma única thread pode executar esse ciclo. Distribuir os módulos entre threads
concorrentes exige revisar seus contratos e a sincronização do estado e do I2C.

## Plataformas e versões

O recorte da aplicação compreende ESP32, STM32F411, STM32F407 e nRF52840,
conforme a proposta do framework. O compartilhamento dos fontes não substitui
a conformidade dos portes nem os testes funcionais e de temporização em cada
placa.

Os projetos integradores devem fixar o mesmo commit deste submódulo e versões
compatíveis dos componentes e das interfaces da HAL. O nome `framework_app`
identifica o repositório remoto; `src/app` identifica seu caminho no firmware.

O componente `telemetry` deve fornecer `cmp_telemetry_resync()`. Após uma
transmissão parcial, a aplicação descarta o registro truncado e envia CRLF antes
de liberar os próximos registros. Falhas nesse delimitador são repetidas com
intervalo de 1000 ms, sem reenviar o prefixo do registro descartado. O receptor
deve ignorar linhas vazias ou incompletas. O estado de recuperação é mantido até
o delimitador ser transmitido; reinicializar a aplicação pressupõe uma nova
fronteira de fluxo estabelecida pela integração.
