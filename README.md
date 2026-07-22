# monitor-inclinacao-tronco

Ferramenta web **standalone e independente** para coleta de dados biomecânicos: usa o acelerômetro do celular (`DeviceMotionEvent.accelerationIncludingGravity`) para monitorar, em tempo real, a inclinação do tronco de um atleta cadeirante durante atividade física.

> Este projeto não tem nenhuma relação com sistemas de navegação de cadeira de rodas, controle diferencial, firmware embarcado, ROS, ESP32 ou qualquer outro repositório. É uma ferramenta isolada, sem dependências externas.

## Como usar

1. Prenda o celular ao peito do atleta, em **portrait**, com a tela voltada para fora.
2. Abra a página (precisa ser servida via **HTTPS** ou `localhost` — sensores de movimento não funcionam em origens inseguras).
3. Toque em **"Ativar sensores"** e conceda a permissão quando solicitado (obrigatório no iOS).
4. Com o celular já na posição de referência do atleta, toque em **"Zerar referência"** para compensar qualquer desalinhamento na fixação.
5. Toque em **"Iniciar gravação"** para começar a registrar as amostras. Um cronômetro e um gráfico rolante (últimos ~15s) mostram os dados em tempo real.
6. Toque em **"Parar gravação"** ao final e depois em **"Exportar CSV"** para baixar o arquivo com as colunas `tempo_s, flexao_extensao_graus, inclinacao_lateral_graus`.
7. Use **"Descartar e recomeçar"** a qualquer momento para limpar o log e gravar novamente.

Nada é salvo automaticamente: todo o estado vive apenas na memória da aba enquanto ela estiver aberta.

## Características técnicas

- Um único arquivo (`index.html`) autocontido: HTML, CSS e JavaScript puros, sem build step, sem framework, sem dependências externas.
- Sem `localStorage`/`sessionStorage`.
- Interface 100% em português do Brasil, mobile-first, alto contraste, com foco visível e respeito a `prefers-reduced-motion`.
- Duas silhuetas SVG minimalistas (visão lateral e frontal) que se inclinam conforme `beta` (flexão/extensão) e `gamma` (inclinação lateral), cada uma com uma linha de referência vertical fixa.
- Gráfico rolante desenhado em `<canvas>`, sem bibliotecas externas.

### Por que `DeviceMotionEvent` e não `DeviceOrientationEvent`?

Com o celular preso ao peito em portrait (ou seja, na vertical), os ângulos `beta`/`gamma` do `DeviceOrientationEvent` sofrem um problema clássico de "gimbal lock": a decomposição em ângulos de Euler que o navegador usa degenera perto de `beta ≈ 90°`, e a leitura de inclinação lateral (`gamma`) vira ruído sem sentido — mesmo com o app parado, os números saltam. `beta` continua estável nessa mesma condição, por isso a leitura de flexão/extensão não apresentava esse sintoma.

Por isso o app calcula os dois ângulos diretamente do vetor de gravidade bruto (`accelerationIncludingGravity`, em `x/y/z`) usando `atan2`, que não tem esse ponto de degenerescência:

- `flexão/extensão = atan2(z, y)`
- `inclinação lateral = atan2(x, y)`

Como o acelerômetro bruto é mais ruidoso que a orientação já fundida pelo sistema operacional (especialmente durante atividade física), o app aplica um filtro passa-baixa leve (constante `SUAVIZACAO`) sobre os dois ângulos antes de exibir/gravar.

### Ajustes finos

No topo do bloco `<script>` do `index.html` existem constantes que podem ser calibradas conforme o protocolo de uso:

- `SINAL_BETA` / `SINAL_GAMMA`: invertem o sentido da leitura (afeta o número exibido/gravado e o giro da silhueta juntos), caso pareça invertido no aparelho usado. Valor padrão atual: `-1` para os dois.
- `SUAVIZACAO`: força do filtro passa-baixa (0–1) aplicado às leituras do acelerômetro; menor = mais suave e mais lento.
- `FAIXA_BETA_ATENCAO`, `FAIXA_BETA_ALERTA`, `FAIXA_GAMMA_ATENCAO`, `FAIXA_GAMMA_ALERTA`: limites (em graus) que definem as faixas de cor neutro/atenção/alerta.
- `INTERVALO_MIN_AMOSTRA_MS`: taxa de amostragem do log de gravação (padrão ~20 Hz).

## Publicar no GitHub Pages

Passo a passo exato para colocar o link no ar:

1. Crie um repositório novo no GitHub chamado `monitor-inclinacao-tronco` (pode ser público ou privado — Pages funciona nos dois em contas com esse recurso liberado).
2. No terminal, dentro desta pasta, aponte o repositório local para o remoto e envie o código:
   ```bash
   git remote add origin https://github.com/SEU_USUARIO/monitor-inclinacao-tronco.git
   git branch -M main
   git push -u origin main
   ```
3. No GitHub, entre no repositório e vá em **Settings → Pages**.
4. Em **Build and deployment**, selecione a origem **"Deploy from a branch"**.
5. Em **Branch**, escolha `main` e a pasta `/ (root)`.
6. Clique em **Save**.
7. Aguarde alguns instantes; o GitHub Pages vai publicar o site e o link ficará disponível em:
   ```
   https://SEU_USUARIO.github.io/monitor-inclinacao-tronco/
   ```

Como o GitHub Pages serve tudo via HTTPS, o pedido de permissão de sensores (`DeviceMotionEvent.requestPermission`, necessário no iOS) funciona normalmente a partir desse link.

## Formato do CSV exportado

```
tempo_s,flexao_extensao_graus,inclinacao_lateral_graus
0.000,0.42,-1.10
0.052,0.55,-0.98
...
```

- `tempo_s`: tempo relativo ao início da gravação, em segundos.
- `flexao_extensao_graus`: inclinação frente-trás do tronco (a partir de `beta`), já corrigida pela referência zerada.
- `inclinacao_lateral_graus`: inclinação lateral do tronco (a partir de `gamma`), já corrigida pela referência zerada.
