<p align="center">
	<h1 align="center"><b>Crawl Inference</b></h1>
<p align="center">
Infraestructura definitiva de Web scraping para la era de la IA, bajo licencia MIT</p>
</p>

> ⚠️ Este repositorio se encuentra aún en etapa temprana de desarrollo, por lo que podrías encontrar errores.  
> Si encuentras algún problema, no dudes en abrir un issue o enviar un PR; las contribuciones son muy bienvenidas. Gracias 💜


## Inicio Rápido

```bash
git clone https://github.com/0xlakshan/crawl-inference.git
cd ./crawl-inference
npm install
```

## Uso

### Cliente SDK

```typescript
import { Scraper } from "scrape-kit";
import { z } from "zod";

const ProductSchema = z.object({
  name: z.string().describe("Product title text"),
  price: z.number().describe("Numeric price without currency"),
  image: z.string().url().describe("Main product image URL"),
});

const scraper = new Scraper();

// Estimar el uso de tokens antes del scraping
const usage = await scraper.getTokenUsage("https://example.com/products", {
  prompt: "return me top five products",
  schema: ProductSchema,
  model: "gemini-2.0-flash-exp",
  waitFor: 2000,
  timeout: 30000,
});

console.log(usage.tokens); // { prompt, schema, total }
console.log(usage.estimatedCost); // Desglose de costos

// Scrapear con salida estructurada
const result = await scraper.scrape("https://example.com/products", {
  prompt: "return me top five products",
  schema: ProductSchema,
  model: "gemini-2.0-flash-exp",
  output: "json",
  waitFor: 2000,
  timeout: 30000,
  postProcess: (data) => data,
});

console.log(result.data);
```

## API

### `new Scraper()`

Crea una nueva instancia del scraper. El SDK se conecta al servidor de la API (por defecto: `http://localhost:3000`, configurable mediante la variable de entorno `SCRAPE_API_URL`).

### `scraper.scrape(url, options)`

Extrae datos de una URL y devuelve información estructurada.

**Opciones:**
- **prompt** - Indica a la IA qué deseas extraer de la página. ¡Sé específico! Por ejemplo: "Obtén todos los nombres y precios de los productos" o "Extrae el título del artículo, el autor y la fecha de publicación". Cuanto más claro sea tu prompt, mejores serán los resultados.

- **schema** - Define la forma de tus datos usando Zod. Es como un plano que le dice al scraper exactamente qué campos extraer y qué tipo debe ser cada campo (string, number, URL, etc.). La IA validará los datos extraídos contra este esquema para asegurar que obtengas una salida limpia y estructurada.

- **model** - Qué modelo de Google Gemini utilizar para la extracción. Diferentes modelos tienen diferentes capacidades y costos. Por ejemplo, "gemini-2.0-flash-exp" es rápido y rentable para la mayoría de las tareas de scraping.

- **output** - Cómo deseas que se formatee la información. Actualmente solo soporta "json" (que es el valor por defecto), por lo que normalmente puedes omitir esta opción.

- **waitFor** - Tiempo de espera (en milisegundos) después de que la página cargue antes de iniciar la extracción. Útil para páginas con contenido dinámico que carga mediante JavaScript. Por ejemplo, establece 2000 para esperar 2 segundos a que aparezcan las animaciones o el contenido cargado perezosamente (lazy-loaded).

- **timeout** - Tiempo máximo (en milisegundos) a esperar para que la página cargue antes de desistir. Por defecto es de 30 segundos (30000ms). Auméntalo si estás extrayendo datos de páginas de carga lenta.

- **postProcess** - Una función callback para transformar o limpiar los datos extraídos antes de que sean devueltos. Recibe los datos extraídos brutos como entrada y devuelve tu versión modificada. Útil para tareas como formatear fechas, convertir monedas o filtrar elementos no deseados.

**Retorna:** `Promise<ScrapeResult<T>>`

```typescript
{
  url: string;
  data: T; // Analizado y validado contra el esquema
  format: "json";
}
```

### `scraper.getTokenUsage(url, options)`

Estima el uso de tokens y el costo antes de realizar el scraping.

| Opción    | Tipo        | Requerido | Descripción                        |
| --------- | ----------- | -------- | ---------------------------------- |
| `prompt`  | `string`    | Sí       | Prompt de extracción de la IA      |
| `schema`  | `z.ZodType` | Sí       | Esquema de Zod para la salida      |
| `model`   | `string`    | No       | Modelo (defecto: gemini-2.0-flash-exp) |
| `waitFor` | `number`    | No       | Tiempo de espera en ms tras carga  |
| `timeout` | `number`    | No       | Timeout de carga (defecto: 30000)  |

**Retorna:** `Promise<TokenUsageResult>`

```typescript
{
  url: string;
  tokens: {
    prompt: number;
    schema: number;
    total: number;
  };
  estimatedCost?: {
    inputCostPer1M: number;
    outputCostPer1M: number;
    estimatedInput: string;
    estimatedOutput: string;
  };
}
```

## Servidor

Inicia el servidor de la API:

```bash
npm run serve
# o
GOOGLE_API_KEY=xxx npx tsx packages/api/src/index.ts
```

El servidor se ejecuta en el puerto 3000 por defecto (configurable mediante la variable de entorno `PORT`).

### Endpoints

#### `POST /scrape`

Extrae datos de una URL y devuelve información estructurada.

**Request:**
```json
{
  "url": "https://example.com",
  "prompt": "extract products",
  "model": "gemini-2.0-flash-exp",
  "schema": {
    "type": "object",
    "properties": { "name": { "type": "string" } }
  },
  "output": "json",
  "waitFor": 2000,
  "timeout": 30000
}
```

**Response:**
```json
{
  "success": true,
  "data": { "name": "Product Name" }
}
```

#### `POST /token-usage`

Estima el uso de tokens y el costo.

**Request:**
```json
{
  "url": "https://example.com",
  "prompt": "extract products",
  "model": "gemini-2.0-flash-exp",
  "schema": {
    "type": "object",
    "properties": { "name": { "type": "string" } }
  },
  "waitFor": 2000,
  "timeout": 30000
}
```

**Response:**
```json
{
  "success": true,
  "tokens": {
    "prompt": 150,
    "schema": 50,
    "total": 200
  },
  "estimatedCost": {
    "inputCostPer1M": 0.075,
    "outputCostPer1M": 0.30,
    "estimatedInput": "$0.000015",
    "estimatedOutput": "$0.000060"
  }
}
```

#### `GET /health`

Endpoint de verificación de estado (health check).

**Response:**
```json
{
  "status": "ok"
}
```

## Variables de Entorno

- `GOOGLE_API_KEY` - Requerido para el servidor de la API
- `SCRAPE_API_URL` - Endpoint de la API para el cliente SDK (defecto: `http://localhost:3000`)
- `PORT` - Puerto del servidor de la API (defecto: `3000`)

## Desarrollo

```bash
# Construir SDK
npm run build

# Modo watch
npm run dev

# Ejecutar tests
npm test

# Cobertura de tests
npm run test:coverage

# Lint
npm run lint
```
