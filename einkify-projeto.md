# E-inkify

Filtro de tela para Android voltado à **leitura com iluminação absolutamente mínima**, principalmente navegação na internet. A ideia central é converter o conteúdo da tela em uma imagem de 1 bit (pixel aceso ou apagado), com tinta clara e de baixa intensidade sobre preto, aproveitando que, em telas OLED, pixel preto fica apagado.

Documento de registro: 28 de outubro de 2026.

---

## 1. Objetivo e premissas

- **Uso:** ler no escuro, à noite, com a menor emissão de luz possível.
- **Tela-alvo:** OLED. Em LCD a luz de fundo fica ligada de qualquer forma e o ganho cai bastante.
- **Saída:** 1 bit, sem tons intermediários. A cor e a intensidade da "tinta" são ajustáveis: cinza, âmbar ou vermelho.
- **Prioridade:** texto legível com o mínimo de pixels acesos e sem borrão.

## 2. Por onde passamos

### 2.1 Proposta inicial: arestas por contraste entre células

Primeira ideia: dividir a tela em células, calcular um score de contraste nas fronteiras e desenhar a aresta quando o contraste passasse de um limiar. Pontos levantados na discussão:

- Um limiar global é frágil. A mesma diferença absoluta pesa mais em fundo escuro do que em fundo claro.
- O tamanho da célula deve ser pensado em dp, porque texto pequeno some se a célula for maior que o traço.
- Texto vira contorno. Se o traço é mais grosso que a célula, a letra fica oca; se é mais fino, as bordas se fundem.
- Uma grade fixa gera serrilhado em diagonais e curvas.
- Na rolagem, o conteúdo desliza sob a grade e as bordas piscam.
- No Android, o gargalo é a captura da tela, não o algoritmo.

### 2.2 Ajuste de foco para leitura no escuro

- Para texto, um traço sólido e fino lê melhor que um contorno e acende menos pixels.
- No escuro, o olho precisa de traço mais grosso e letra maior que o normal.
- A intensidade deve ser reduzida na própria cor de saída. Em OLED isso economiza de verdade, ao contrário de um overlay escurecedor.
- Um canal vermelho ou âmbar preserva a adaptação ao escuro, mas fica difícil de ler em luz muito baixa. Vale testar.
- Para navegação, filtrar dentro do navegador evita os problemas de captura.

### 2.3 Protótipo E-inkify (calibrador de leitura)

Página web que calibra cor, intensidade (em escala logarítmica, a partir de 0,2% do máximo), fonte, tamanho, peso do traço, entrelinha, espaçamento entre letras e tratamento de imagens. Ela:

- mede a fração da tela coberta por texto e estima a luz emitida em relação a uma tela branca;
- oferece um teste em tela cheia;
- gera um favorito (bookmarklet) que aplica o mesmo estilo em qualquer site no Chrome do Android.

### 2.4 Pixel do CSS × pixel físico

O `px` do CSS não é o pixel físico. A relação entre os dois é o `devicePixelRatio`, que fica entre 2,6 e 3,5 nos Android atuais e muda com o zoom. Para desenhar no pixel físico exato:

- usar `ResizeObserver` com `device-pixel-content-box`;
- usar um canvas com o tamanho físico real e `image-rendering: pixelated`;
- escrever direto no buffer de pixels, sem antialiasing.

### 2.5 Protótipo Pixel Exato

Canvas 1:1 com os pixels físicos do display. Um xadrez de 1 px aparece como cinza uniforme, sem borrão. Tem caneta, borracha, texto 1 bit, padrões de teste e lupa 8×. Sobre ele foram implementados os dois modos descritos abaixo.

## 3. Modo 1: arestas de grade (descontinuado)

> Removido do Pixel Exato em 28/10/2026. Mantido aqui só como registro.

**Geometria.** As coordenadas `(x, y)` contam a partir do canto superior esquerdo `(0, 0)`, e cada ponto é um pixel.

- Célula quadrada de lado `C` pixels (padrão 3). Também foi testada a "Grade 1", com células de 1 px.
- Aresta com 1 px de largura e `C + 2` px de comprimento, porque arestas vizinhas compartilham o pixel do vértice.
- O passo da grade é `P = C + 1`.
- Pixel de aresta: `x % P == 0` ou `y % P == 0`. Vértice: as duas condições ao mesmo tempo.
- A célula `(i, j)` ocupa `x` de `P·i + 1` a `P·i + C` e `y` de `P·j + 1` a `P·j + C`.

**Regra.**

1. Score do pixel: `(R + G + B) / 768`. O valor 768 é 256 × 3, então o branco puro dá 765/768 ≈ 0,996. A opção 765 faz o branco valer exatamente 1.
2. Score da célula: média dos scores dos pixels internos. Os pixels sob as arestas não entram na conta.
3. A aresta entre duas células adjacentes **liga** se `|score_A − score_B| > 0,3`. Com 0,3 ou menos, fica **desligada**. Não há estado intermediário.
4. O pixel de vértice só acende se as 4 arestas que o formam estiverem acesas. Esse número é configurável de 4 a 1.
5. As arestas da borda da tela têm uma célula só e ficam sempre desligadas.

**Resultado.** Funciona, mas a saída fica presa à malha. Diagonais viram escada com passo de 4 px, a regra de 4 arestas deixa contornos picotados, o interior das células nunca acende e letras viram contornos.

## 4. Modo 2: contraste com a vizinhança

**Regra.**

1. A imagem original é dividida em **grupos** quadrados de `n × n` pixels, sem espaço entre eles. Com `n = 1`, cada pixel é avaliado individualmente.
2. Score do grupo: média dos scores dos seus pixels, pela mesma conta do Modo 1.
3. **Vizinhança em anéis.** Com `R` anéis, o grupo é comparado com todos os grupos do quadrado `(2R+1) × (2R+1)` ao seu redor, sem contar ele mesmo. Exemplo com `R = 1`: o pixel `(5, 5)` é comparado com `(4,4)`, `(5,4)`, `(6,4)`, `(4,5)`, `(6,5)`, `(4,6)`, `(5,6)` e `(6,6)`. Com `R = 2`, entra a próxima camada externa.

   | Anéis | Janela | Vizinhos |
   |---|---|---|
   | 1 | 3×3 | 8 |
   | 2 | 5×5 | 24 |
   | 3 | 7×7 | 48 |
   | R | (2R+1)² | (2R+1)² − 1 |

4. **Medida do contraste (Δ da média):** a diferença entre o grupo e a média dos vizinhos, equivalente a uma binarização adaptativa por média local, no estilo Bradley ou Sauvola. A opção *maior Δ* (gradiente morfológico) foi testada e removida.
5. **Polaridade:** *qualquer* (valor absoluto), *mais claro* (só acende o lado claro de uma borda) ou *mais escuro*.
6. O grupo acende inteiro se o contraste for maior que o limiar.
7. Nas bordas da tela entram só os vizinhos que existem. A sobra da tela que não forma um grupo completo fica apagada.
8. **AA diagonal (opcional).** Uma passada 3×3 sobre o resultado, tratando cada grupo como um pixel. Com a vizinhança

   ```
   A B C
   D E F
   G H I
   ```

   um grupo aceso (`E`) continua aceso. Um grupo apagado recebe meia-tinta se `D` e `B` estiverem acesos com `A` apagado, ou `B` e `F` com `C` apagado, ou `F` e `H` com `I` apagado, ou `D` e `H` com `G` apagado. Exigir o canto apagado limita a meia-tinta aos degraus de traço fino e preserva cantos internos de áreas cheias e miolos de letras. A meia-tinta é a cor da tinta com metade da luz emitida (cada canal sRGB a cerca de 73%), e a cor da tinta é escolhida num seletor.

**Por que funciona melhor que o Modo 1.**

- Não impõe grade. Com `n = 1` ou `2`, o pixel aceso fica onde está o detalhe da imagem.
- Os 8 vizinhos ou mais tornam a detecção quase igual em todas as direções, e as diagonais saem conectadas.
- Com polaridade, ele acende o traço inteiro, e não só o contorno.
- Reproduz técnicas consagradas de digitalização de documentos.

**Implementação no protótipo.** A média usa imagem integral. O resultado foi conferido contra uma contagem direta, vizinho a vizinho, em 200 combinações de parâmetros, sem nenhuma divergência.

**Configuração inicial sugerida para texto:** grupo 1, anéis entre 3 e 5 (um pouco acima da espessura do traço) e polaridade "mais claro".

**Limitações conhecidas.**

- Traços mais grossos que a vizinhança ficam ocos. A solução é aumentar os anéis.
- Com grupo de 1 px, ruído de JPEG e textura de foto passam do limiar.
- Com `n > 1`, a rolagem faz as bordas piscarem.
- Acende áreas, então pode gastar mais luz que linhas finas. A contagem de grupos acesos serve de referência.

## 5. Como levar para o Android

### Rota 1 (recomendada): shader na própria View

A partir do Android 13 (API 33), `View.setRenderEffect(RenderEffect.createRuntimeShaderEffect(...))` aplica um shader AGSL ao conteúdo já renderizado de uma View. Aplicado a uma WebView, isso cria um navegador próprio com o Modo 2 rodando na GPU.

- Não há captura de tela, consentimento, notificação fixa nem problema com FLAG_SECURE.
- Não há realimentação, porque o shader lê o conteúdo original.
- Só redesenha quando o conteúdo muda.
- Desempenho: máximo, mínimo e média são separáveis. Encadeando duas passagens com `RenderEffect.createChainEffect`, o custo cai de (2R+1)² para 2(2R+1) leituras por pixel.

Esboço do shader, não testado:

```glsl
uniform shader content;
uniform float thr;   // contraste
uniform float R;     // anéis
uniform half3 ink;   // cor e intensidade de saída

float lum(float2 p) { return dot(float3(content.eval(p).rgb), float3(1.0/3.0)); }

half4 main(float2 p) {
  float s = lum(p), sum = 0.0, cnt = 0.0;
  for (int dy = -6; dy <= 6; dy++) {
    for (int dx = -6; dx <= 6; dx++) {
      float2 d = float2(dx, dy);
      if (max(abs(d.x), abs(d.y)) <= R && (dx != 0 || dy != 0)) {
        sum += lum(p + d); cnt += 1.0;
      }
    }
  }
  return half4(ink * half(step(thr, s - sum / cnt)), 1.0);
}
```

### Rota 2: tela inteira, para qualquer app

O problema central: a saída é opaca e cobre a tela, então uma captura do display veria só o próprio overlay. As saídas conhecidas são parciais e ainda precisam ser verificadas num aparelho:

- **Captura de um único app (Android 14 ou mais recente):** o MediaProjection grava só a janela escolhida, sem o overlay. Exige consentimento a cada sessão e funciona com um app por vez.
- **`takeScreenshotOfWindow` do serviço de acessibilidade (API 34):** exclui a própria janela, mas a frequência é limitada a poucos quadros por segundo.

### Rota 3: root ou ROM própria

Um filtro no compositor (SurfaceFlinger) resolveria tudo, mas não está disponível para apps comuns.

## 6. Planos futuros

1. **App navegador E-inkify** em Kotlin: WebView com o shader AGSL do Modo 2, parâmetros iguais aos do Pixel Exato e tela de ajuste com cor e intensidade do calibrador.
2. **Otimização do shader** com passagens separáveis encadeadas, medindo o tempo de quadro em 120 Hz.
3. **Separar o tamanho do grupo do raio da vizinhança** em escala fina, para texto grande e títulos em negrito.
4. **Histerese:** um limiar alto para acender e um baixo para manter aceso quando um vizinho está aceso, para reduzir ruído e falhas no traço.
5. **Piso de variância local**, no estilo Sauvola, para não acender textura de foto.
6. **Suavização das bordas** na conversão de texto com antialiasing, que ficou planejada desde o início.
7. **Leitor de EPUB e PDF** com o mesmo shader.
8. **Experimento de tela inteira** pela rota 2, depois que o navegador estiver estável.
9. **Validação no escuro real:** comparar cinza, âmbar e vermelho, e medir a luz emitida com a contagem de pixels acesos.

## 7. Artefatos

- **Pixel Exato:** canvas 1:1 em pixels físicos com o Modo 2, parâmetros ajustáveis, inspeção por toque e lupa. Arquivo `pixel-exato.html`.
- **E-inkify (calibrador):** prévia de leitura em luz mínima e gerador do favorito para o Chrome do Android.
- **Nome:** "E-inkify". Antes de usar publicamente, vale pesquisar no INPI e na Play Store, por causa da marca "E Ink".
