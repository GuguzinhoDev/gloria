# GlorIA — Personal Stylist com IA

**GlorIA** é uma API de busca semântica e consultoria de moda construída para e-commerce de calçados e bolsas. Ela entende linguagem natural, lembra do contexto da conversa e adiciona produtos direto no carrinho.

Em vez de digitar "scarpin preto 38" em uma barra de pesquisa, a cliente escreve *"preciso de algo elegante para uma reunião importante"* — e a GlorIA encontra, explica por que cada produto faz sentido, e pergunta se quer adicionar ao carrinho.

---

## O que a GlorIA sabe fazer

- **Entende o que você quer dizer**, não só o que você escreveu. "Algo para o verão na praia" encontra sandálias e rasteiras, mesmo que a palavra "sandália" nunca apareça na busca.
- **Lembra da conversa**. Se você disse que gosta de estilo elegante na primeira mensagem, a segunda busca já herda esse contexto.
- **Conhece sinônimos do mundo fashion**. Vinho, bordô, marsala e oxblood são a mesma cor. Sapatilha, flat e bailarina são o mesmo produto. Mule, tamanco e slide também.
- **Sugere complementos** de forma natural. Encontrou o scarpin perfeito? Ela sugere uma bolsa tiracolo que combina.
- **Adiciona ao carrinho da VTEX** com um clique, sem sair da conversa.
- **Mostra tamanhos reais**, consultando o estoque da VTEX em tempo real.

---

## Como o motor funciona

O coração da GlorIA é um **motor de busca híbrido** que combina três abordagens:

**1. Busca vetorial semântica** — cada produto é transformado em um vetor matemático que representa seu *significado*. Quando a cliente busca, a GlorIA compara o significado da busca com o significado de cada produto, não só as palavras.

**2. Busca BM25** — o algoritmo clássico de relevância por palavras-chave, que garante que termos específicos como um nome de modelo ou referência exata não se percam.

**3. Fusão RRF** — os dois rankings são combinados usando Reciprocal Rank Fusion, que dá mais peso para produtos que aparecem bem nos dois métodos ao mesmo tempo.

O resultado dos três passa por um **re-ranking final via Gemini**, que garante diversidade (sem mostrar seis variações do mesmo sapato) e relevância contextual real.

Se o ChromaDB ou o BM25 não estiverem disponíveis, a GlorIA cai graciosamente para um **scoring heurístico de 7 camadas** que funciona sem dependências externas.

---

## Instalação

```bash
# Clone o repositório
git clone https://github.com/seu-usuario/gloria-api
cd gloria-api

# Crie o .env
cp .env.example .env
# Edite com sua GEMINI_API_KEY e VTEX_BASE_URL

# Instalação mínima — funciona já, sem vetorial
pip install fastapi uvicorn httpx python-dotenv google-genai python-multipart Pillow

# Para ativar BM25 (recomendado)
pip install rank-bm25

# Para ativar busca vetorial semântica (melhor qualidade)
pip install chromadb

# Para ativar memória conversacional entre sessões
pip install redis
```

Coloque seu `produtos.json` na raiz e suba:

```bash
uvicorn gloria_api:app --reload --port 8000
```

Se for a primeira vez com o ChromaDB instalado, indexe o catálogo:

```bash
curl -X POST http://localhost:8000/admin/indexar
```

Isso leva alguns minutos e só precisa ser feito uma vez (ou quando o catálogo mudar).

---

## Endpoints

| Método | Rota | Para que serve |
|--------|------|----------------|
| `POST` | `/consultar` | Busca por texto ou imagem, retorna produtos + consultoria |
| `POST` | `/adicionar-carrinho` | Adiciona SKU ao carrinho VTEX |
| `GET` | `/carrinho` | Mostra o carrinho atual da sessão |
| `GET` | `/tamanhos/{id}` | Tamanhos disponíveis em estoque real |
| `GET` | `/analytics/summary` | Resumo de uso e performance |
| `POST` | `/admin/indexar` | Indexa/reindexar o catálogo no ChromaDB |
| `DELETE` | `/admin/cache` | Limpa caches em memória |
| `GET` | `/health` | Status de todos os componentes |

### Exemplo de busca

```bash
curl -X POST http://localhost:8000/consultar \
  -F "texto=quero um mule elegante para o trabalho, prefiro preto ou nude" \
  -F "session_id=usuario_123"
```

Resposta:
```json
{
  "status": "sucesso",
  "interpretacao": {
    "ocasiao": "trabalho",
    "tipo_produto": "mule",
    "cores": ["preto", "nude"],
    "estilo_cliente": "elegante",
    "nivel_decisao": "indecisa"
  },
  "consultoria": "Para o trabalho, o Mule Verniz Preto é uma escolha certeira — o verniz dá aquele acabamento refinado sem esforço. O Mule Nude Couro vai com praticamente qualquer roupa de alfaiataria. Qual é o estilo do seu escritório, mais clássico ou moderno?",
  "produtos": [...],
  "proxima_acao": "ajudar_decidir"
}
```

### Exemplo de add to cart

```bash
curl -X POST http://localhost:8000/adicionar-carrinho \
  -H "Content-Type: application/json" \
  -d '{"id_sku": "12042034", "quantidade": 1, "session_id": "vtex_session_aqui"}'
```

---

## Variáveis de ambiente

| Variável | Obrigatória | Padrão |
|----------|:-----------:|--------|
| `GEMINI_API_KEY` | ✅ | — |
| `VTEX_BASE_URL` | ✅ | `https://www.constance.com.br` |
| `GEMINI_MODEL` | ❌ | `gemini-2.0-flash` |
| `EMBED_MODEL` | ❌ | `text-embedding-004` |
| `REDIS_URL` | ❌ | `redis://localhost:6379` |
| `SESSION_TTL_SECONDS` | ❌ | `1800` (30 minutos) |
| `CHROMA_PATH` | ❌ | `./vector_db` |
| `PRODUTOS_JSON` | ❌ | `produtos.json` |

---

## Modos de operação

A GlorIA se adapta ao que estiver disponível no ambiente. Você pode conferir qual modo está ativo em `/health`:

```
🟢 Híbrido  — ChromaDB + BM25 + RRF + Re-rank Gemini  (qualidade máxima)
🟡 BM25     — só palavras-chave + Re-rank Gemini       (bom)
🟡 Vetorial — só semântico + Re-rank Gemini            (bom)
🔴 Heurístico — 7 camadas de score + Re-rank Gemini    (funciona sempre)
```

---

## Estrutura do projeto

```
gloria_api.py          — arquivo único, tudo aqui (para testes e deploy simples)

# Versão modular (para GitHub e times)
main.py                — entrypoint
config.py              — variáveis de ambiente
search/
  lexicon.py           — léxico fashion: sinônimos, cores, ocasiões, materiais
  engine.py            — motor híbrido completo
  memory.py            — memória de sessão Redis
vtex/
  catalog.py           — carregamento e cache do catálogo
  stock.py             — tamanhos reais da VTEX
  cart.py              — add to cart (OrderForm API)
```

---

## O léxico fashion

A GlorIA tem um dicionário próprio de moda brasileira, construído à mão, que cobre:

- **19 famílias de cores** com todos os tons e sinônimos (vinho = bordô = marsala = burgundy)
- **10 tipos de produto** com variações (sapatilha = flat = bailarina = ballerina)
- **7 ocasiões** mapeadas para os tipos ideais e o que evitar (praia evita salto e couro)
- **8 grupos de estilo** com termos associados (elegante = sofisticado = refinado = chique)
- **8 grupos de material** (couro = leather = napa = verniz)

Isso significa que a busca funciona bem mesmo quando a cliente usa gírias, estrangeirismos ou descrições vagas.

---

## Analytics

Toda consulta e add-to-cart é logada em `analytics.jsonl`. O endpoint `/analytics/summary` agrega esses dados em tempo real e expõe para o dashboard. Sem banco de dados adicional.

---

## Tecnologias

- **FastAPI** — API assíncrona
- **Google Gemini** — interpretação de intenção, embeddings e re-ranking
- **ChromaDB** — banco vetorial local (opcional)
- **rank-bm25** — busca por palavras-chave (opcional)
- **Redis** — memória de sessão (opcional)
- **VTEX** — catálogo, estoque e carrinho
