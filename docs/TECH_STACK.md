# Stack Tecnológico Detallado

## Decisiones Técnicas y Justificaciones

### Por qué Next.js 14

**Ventajas**:
- ✅ **Server Components**: Reducen bundle size del cliente
- ✅ **App Router**: Routing moderno y nested layouts
- ✅ **API Routes**: Backend y frontend en un solo proyecto
- ✅ **Image Optimization**: Automática con next/image
- ✅ **TypeScript**: First-class support
- ✅ **Vercel Deploy**: Deployment trivial
- ✅ **SEO**: SSR/SSG out of the box

**Alternativas consideradas**:
- ❌ **Remix**: Menos maduro, menor ecosistema
- ❌ **SvelteKit**: Menor pool de desarrolladores
- ❌ **Pure React + Express**: Más boilerplate, dos proyectos separados

---

### Por qué Solana vs Polygon

**Ventajas de Solana**:
- ✅ **Costos**: $0.00025 por transacción (vs $0.001 Polygon)
- ✅ **Velocidad**: 400ms confirmación (vs 2s Polygon)
- ✅ **Throughput**: 65,000 TPS (vs 7,000 Polygon)
- ✅ **Developer Experience**: Anchor framework es excelente
- ✅ **No gas tokens**: Usuario no necesita SOL (con Squads)

**Desventajas**:
- ⚠️ **Network Outages**: Historia de downtime
- ⚠️ **Menor Adoption**: Menos DApps que Ethereum/Polygon
- ⚠️ **Learning Curve**: Rust vs Solidity

**Mitigación**:
- Diseño multi-chain en roadmap (Fase 2)
- Architecture permite cambiar blockchain sin rehacer frontend
- Solana ha mejorado estabilidad significativamente en 2024

**Decisión**: Solana para MVP, multi-chain después

---

### Anchor Framework

**Por qué Anchor**:
```rust
// Sin Anchor (Solana vanilla)
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    // 100+ líneas de boilerplate
    // Validación manual de cuentas
    // Deserialización manual
    // Manejo de errores verboso
}

// Con Anchor
#[program]
pub mod my_program {
    pub fn my_instruction(ctx: Context<MyContext>, data: u64) -> Result<()> {
        // Lógica de negocio directamente
        ctx.accounts.my_account.data = data;
        Ok(())
    }
}
```

**Beneficios**:
- Reduce código en ~70%
- Type-safe IDL generation
- Built-in account validation
- Better error handling
- Test framework incluido

---

### Supabase vs Alternativas

**Por qué Supabase**:
- ✅ **Postgres**: SQL robusto y familiar
- ✅ **Real-time**: Subscriptions built-in
- ✅ **Auth**: Sistema de autenticación incluido
- ✅ **Storage**: File storage integrado
- ✅ **Row Level Security**: Seguridad granular
- ✅ **Free Tier**: Generoso para MVP
- ✅ **TypeScript**: Generated types

**Alternativas**:
- **Firebase**: NoSQL (menos adecuado para queries complejas)
- **MongoDB Atlas**: Requiere gestión separada de auth
- **PlanetScale**: Solo DB, no real-time
- **Railway**: Menos features out-of-box

**Rol en la Arquitectura**:
```
Blockchain: Source of truth (immutable)
      ↓
Supabase: Cache + Search + Analytics
      ↓
Cliente: Fast queries
```

---

### IPFS via Pinata

**Por qué IPFS**:
- ✅ **Decentralized**: No single point of failure
- ✅ **Content Addressing**: CID garantiza integridad
- ✅ **Permanent**: Con pinning services
- ✅ **Standard**: Ecosystem robusto

**Por qué Pinata**:
- ✅ **Confiable**: 99.9% uptime
- ✅ **Performance**: CDN global
- ✅ **Dedicated Gateways**: Rápido para usuarios
- ✅ **Simple API**: Fácil integración
- ✅ **Submarine**: Files privados (si necesario)

**Alternativas**:
- **Web3.Storage**: Gratis pero menos reliable
- **NFT.Storage**: Diseñado para NFTs pequeños
- **Filebase**: Interesante pero menos maduro

**Costo**: ~$20/mes para 1TB storage + 100GB bandwidth

---

### Sharp para Image Processing

```typescript
import sharp from 'sharp';

// Redimensionar manteniendo ratio
await sharp(buffer)
  .resize(2000, 2000, { fit: 'inside' })
  .toBuffer();

// Aplicar watermark
await sharp(buffer)
  .composite([{
    input: watermarkBuffer,
    gravity: 'southeast',
  }])
  .toBuffer();

// Optimizar
await sharp(buffer)
  .jpeg({ quality: 90 })
  .toBuffer();
```

**Por qué Sharp**:
- ✅ **Performance**: Basado en libvips (C)
- ✅ **Memory Efficient**: Streaming
- ✅ **Features**: Resize, composite, format conversion
- ✅ **Cross-platform**: Funciona en Vercel

**Alternativas**:
- ❌ **Jimp**: Pure JS (más lento)
- ❌ **ImageMagick**: Difícil en serverless

---

### TailwindCSS + shadcn/ui

**Por qué Tailwind**:
```tsx
// Traditional CSS
<div className="card">
  <h2 className="card-title">Title</h2>
</div>

// styles.css
.card { padding: 1rem; background: white; ... }
.card-title { font-size: 1.5rem; ... }

// Tailwind
<div className="p-4 bg-white rounded-lg shadow">
  <h2 className="text-2xl font-bold">Title</h2>
</div>
```

**Ventajas**:
- No context switching (HTML ↔ CSS)
- Purge CSS automático (bundle pequeño)
- Responsive design trivial
- Dark mode built-in

**Por qué shadcn/ui**:
- ✅ **No es librería**: Copy-paste components (ownership total)
- ✅ **Customizable**: Tailwind-based
- ✅ **Accessible**: ARIA compliant
- ✅ **Beautiful**: Diseño profesional

**Alternativas**:
- **MUI**: Bundle grande, menos customizable
- **Chakra**: Más opinado
- **Headless UI**: Solo lógica, más trabajo de styling

---

### Zustand para State Management

```typescript
// store/useWalletStore.ts
import create from 'zustand';

interface WalletState {
  connected: boolean;
  publicKey: string | null;
  connect: (publicKey: string) => void;
  disconnect: () => void;
}

export const useWalletStore = create<WalletState>((set) => ({
  connected: false,
  publicKey: null,
  connect: (publicKey) => set({ connected: true, publicKey }),
  disconnect: () => set({ connected: false, publicKey: null }),
}));

// Uso en componente
function MyComponent() {
  const { connected, connect } = useWalletStore();
  // ...
}
```

**Por qué Zustand**:
- ✅ **Simple**: API minimalista
- ✅ **Small**: 1KB
- ✅ **No providers**: No wrapper hell
- ✅ **TypeScript**: Excelente support

**Alternativas**:
- **Redux**: Demasiado boilerplate para MVP
- **Jotai**: Más atómico de lo necesario
- **React Context**: Suficiente para MVP simple, pero Zustand apenas añade complejidad

---

### Solana Wallet Adapter

```typescript
// app/layout.tsx
import { WalletAdapterNetwork } from '@solana/wallet-adapter-base';
import { WalletProvider } from '@solana/wallet-adapter-react';
import { WalletModalProvider } from '@solana/wallet-adapter-react-ui';
import { PhantomWalletAdapter, SolflareWalletAdapter } from '@solana/wallet-adapter-wallets';

export default function RootLayout({ children }) {
  const wallets = [
    new PhantomWalletAdapter(),
    new SolflareWalletAdapter(),
  ];

  return (
    <WalletProvider wallets={wallets} autoConnect>
      <WalletModalProvider>
        {children}
      </WalletModalProvider>
    </WalletProvider>
  );
}
```

**Wallets Soportadas**:
- Phantom (más popular)
- Solflare
- Backpack
- Coinbase Wallet
- Ledger

**UX**:
- Modal automático para selección
- Auto-reconnect
- Mobile support

---

## Dependencias Clave

### package.json

```json
{
  "dependencies": {
    "@coral-xyz/anchor": "^0.29.0",
    "@solana/wallet-adapter-base": "^0.9.23",
    "@solana/wallet-adapter-react": "^0.15.35",
    "@solana/wallet-adapter-react-ui": "^0.9.35",
    "@solana/wallet-adapter-wallets": "^0.19.32",
    "@solana/web3.js": "^1.87.6",
    "@supabase/supabase-js": "^2.39.0",
    "next": "14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "sharp": "^0.33.2",
    "zustand": "^4.5.0",
    "crypto-js": "^4.2.0",
    "exif-parser": "^0.1.12",
    "react-dropzone": "^14.2.3",
    "lucide-react": "^0.323.0",
    "tailwindcss": "^3.4.1",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.1"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^18",
    "typescript": "^5",
    "eslint": "^8",
    "prettier": "^3.2.4",
    "@playwright/test": "^1.41.2"
  }
}
```

---

## Estructura de Datos

### Solana Account Structure

```rust
#[account]
pub struct Certificate {
    pub content_hash: [u8; 32],       // 32 bytes - SHA-256
    pub ipfs_cid: String,             // Variable - IPFS CID
    pub creator: Pubkey,              // 32 bytes - Wallet
    pub timestamp: i64,               // 8 bytes - Unix timestamp
    pub media_type: MediaType,        // 1 byte - Enum
    pub metadata_uri: String,         // Variable - IPFS metadata
    pub authenticity_score: u8,       // 1 byte - 0-100
    pub revoked: bool,                // 1 byte
    pub revoke_reason: Option<String>,// Variable
    pub bump: u8,                     // 1 byte - PDA bump
}

// Total size: ~500 bytes (con strings)
// Rent-exempt: ~0.0035 SOL (~$0.35)
```

### Supabase Schema

```sql
certificates (
    id UUID PRIMARY KEY,
    content_hash TEXT UNIQUE,         -- Indexed
    ipfs_cid TEXT,
    creator_wallet TEXT,              -- Indexed
    media_type TEXT,                  -- 'image' | 'video'
    file_size BIGINT,
    dimensions JSONB,                 -- {width, height, duration?}
    metadata JSONB,                   -- EXIF, custom fields
    authenticity_score INTEGER,
    revoked BOOLEAN,
    solana_signature TEXT,            -- TX hash
    solana_account TEXT,              -- PDA address
    created_at TIMESTAMPTZ,           -- Indexed
    updated_at TIMESTAMPTZ
)

verifications (
    id UUID PRIMARY KEY,
    certificate_id UUID REFERENCES certificates,
    status TEXT,                      -- verified | not_found | altered
    similarity_score INTEGER,
    verifier_ip TEXT,                 -- For analytics
    verified_at TIMESTAMPTZ           -- Indexed
)
```

---

## Performance Considerations

### Image Upload Optimization

```typescript
// Cliente comprime antes de upload
async function optimizeBeforeUpload(file: File): Promise<Blob> {
  const bitmap = await createImageBitmap(file);
  const canvas = document.createElement('canvas');

  // Max 4000px en lado más largo
  const maxSize = 4000;
  const scale = Math.min(1, maxSize / Math.max(bitmap.width, bitmap.height));

  canvas.width = bitmap.width * scale;
  canvas.height = bitmap.height * scale;

  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(bitmap, 0, 0, canvas.width, canvas.height);

  return new Promise((resolve) => {
    canvas.toBlob(resolve, 'image/jpeg', 0.9);
  });
}
```

### Caching Strategy

```typescript
// Verificaciones cacheadas en Redis (Upstash)
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL!,
  token: process.env.UPSTASH_REDIS_TOKEN!,
});

async function verifyWithCache(hash: string) {
  // Check cache
  const cached = await redis.get(`verify:${hash}`);
  if (cached) return cached;

  // Verify from blockchain
  const result = await verifyFromBlockchain(hash);

  // Cache for 1 hour
  await redis.setex(`verify:${hash}`, 3600, result);

  return result;
}
```

### Lazy Loading

```typescript
// Componentes pesados lazy-loaded
import dynamic from 'next/dynamic';

const CertificateViewer = dynamic(
  () => import('@/components/CertificateViewer'),
  { loading: () => <Skeleton /> }
);

const VideoPlayer = dynamic(
  () => import('@/components/VideoPlayer'),
  { ssr: false }
);
```

---

## Seguridad

### Input Validation (Zod)

```typescript
import { z } from 'zod';

const CertifySchema = z.object({
  file: z.instanceof(File)
    .refine((f) => f.size <= 100 * 1024 * 1024, 'Max 100MB')
    .refine(
      (f) => ['image/jpeg', 'image/png', 'video/mp4'].includes(f.type),
      'Invalid file type'
    ),
  metadata: z.object({
    title: z.string().max(200).optional(),
    description: z.string().max(1000).optional(),
  }).optional(),
});

// En API route
export async function POST(request: NextRequest) {
  const formData = await request.formData();

  const validation = CertifySchema.safeParse({
    file: formData.get('file'),
    metadata: JSON.parse(formData.get('metadata') || '{}'),
  });

  if (!validation.success) {
    return NextResponse.json(
      { error: validation.error },
      { status: 400 }
    );
  }

  // Process...
}
```

### Rate Limiting

```typescript
// middleware.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 10 req/min
});

export async function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/api/certify')) {
    const ip = request.ip ?? 'anonymous';
    const { success } = await ratelimit.limit(ip);

    if (!success) {
      return new NextResponse('Too many requests', { status: 429 });
    }
  }

  return NextResponse.next();
}
```

### CORS

```typescript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          { key: 'Access-Control-Allow-Credentials', value: 'true' },
          { key: 'Access-Control-Allow-Origin', value: process.env.ALLOWED_ORIGIN || '*' },
          { key: 'Access-Control-Allow-Methods', value: 'GET,POST,OPTIONS' },
          { key: 'Access-Control-Allow-Headers', value: 'X-CSRF-Token, X-Requested-With, Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, X-Api-Version' },
        ],
      },
    ];
  },
};
```

---

## Testing Strategy

### Unit Tests (Jest)

```typescript
// __tests__/lib/utils/hash.test.ts
import { calculateHash } from '@/lib/utils/hash';

describe('calculateHash', () => {
  it('should generate consistent SHA-256 hash', () => {
    const data = new Uint8Array([1, 2, 3, 4, 5]);
    const hash1 = calculateHash(data);
    const hash2 = calculateHash(data);

    expect(hash1).toBe(hash2);
    expect(hash1).toHaveLength(64); // SHA-256 hex = 64 chars
  });

  it('should generate different hashes for different data', () => {
    const data1 = new Uint8Array([1, 2, 3]);
    const data2 = new Uint8Array([4, 5, 6]);

    expect(calculateHash(data1)).not.toBe(calculateHash(data2));
  });
});
```

### Integration Tests (Playwright)

```typescript
// e2e/certify.spec.ts
import { test, expect } from '@playwright/test';

test('full certification flow', async ({ page }) => {
  await page.goto('/certify');

  // Connect wallet (usando test wallet)
  await page.click('[data-testid="wallet-button"]');
  await page.click('text=Phantom');

  // Upload file
  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('tests/fixtures/test-image.jpg');

  // Certify
  await page.click('text=Certify Media');

  // Wait for success
  await expect(page.locator('text=Media Certified Successfully')).toBeVisible({
    timeout: 30000,
  });

  // Verify certificate ID is shown
  const certId = await page.textContent('[data-testid="certificate-id"]');
  expect(certId).toBeTruthy();
});
```

### Solana Program Tests

```typescript
// tests/media-validator.ts
import * as anchor from '@coral-xyz/anchor';
import { assert } from 'chai';

describe('media-validator', () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.MediaValidator;

  it('certifies media', async () => {
    const contentHash = Array(32).fill(1); // Dummy hash

    const [certificatePda] = PublicKey.findProgramAddressSync(
      [Buffer.from('certificate'), Buffer.from(contentHash)],
      program.programId
    );

    await program.methods
      .certifyMedia(
        contentHash,
        'QmTestCID',
        { image: {} },
        'metadata_uri',
        95
      )
      .accounts({
        certificate: certificatePda,
        creator: provider.wallet.publicKey,
      })
      .rpc();

    const certificate = await program.account.certificate.fetch(certificatePda);

    assert.equal(certificate.authenticityScore, 95);
    assert.equal(certificate.ipfsCid, 'QmTestCID');
  });
});
```

---

## Deployment

### Vercel Configuration

```json
// vercel.json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["iad1"],
  "env": {
    "NEXT_PUBLIC_SOLANA_RPC_URL": "@solana-rpc-url",
    "NEXT_PUBLIC_PROGRAM_ID": "@program-id",
    "SUPABASE_SERVICE_ROLE_KEY": "@supabase-service-key"
  }
}
```

### Solana Program Deployment

```bash
# Build
anchor build

# Deploy to devnet
anchor deploy --provider.cluster devnet

# Verify deployment
solana program show <PROGRAM_ID> --url devnet

# Upgrade (después de cambios)
anchor upgrade target/deploy/media_validator.so \
  --program-id <PROGRAM_ID> \
  --provider.cluster devnet
```

### CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy-vercel:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

---

## Monitoring

### Sentry for Error Tracking

```typescript
// instrumentation.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  environment: process.env.NODE_ENV,
});
```

### Analytics

```typescript
// lib/analytics.ts
import { Analytics } from '@vercel/analytics/react';

export function trackCertification(certificateId: string) {
  Analytics.track('Certification', { certificateId });
}

export function trackVerification(status: string) {
  Analytics.track('Verification', { status });
}
```

---

**Stack optimizado para velocidad de desarrollo y escalabilidad futura.**
