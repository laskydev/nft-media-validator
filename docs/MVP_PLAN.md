# Plan de Implementación MVP

## NFT Media Validator - Minimum Viable Product

### Stack Tecnológico

#### Frontend & Backend
- **Framework**: Next.js 14 (App Router)
- **Lenguaje**: TypeScript
- **Styling**: TailwindCSS + shadcn/ui
- **State Management**: Zustand
- **Forms**: React Hook Form + Zod

#### Blockchain
- **Network**: Solana (Devnet → Mainnet)
- **Smart Contracts**: Rust (Anchor Framework)
- **Wallet**: Solana Wallet Adapter (@solana/wallet-adapter-react)
- **RPC**: Helius o QuickNode

#### Storage
- **Database**: PostgreSQL (Supabase)
- **File Storage**: IPFS (via Pinata o Web3.Storage)
- **Cache**: Redis (Upstash)

#### Processing
- **Image Processing**: Sharp
- **Video Processing**: FFmpeg (vía API)
- **Hash**: crypto-js (SHA-256)

#### Deployment
- **Frontend/Backend**: Vercel
- **Database**: Supabase Cloud
- **IPFS**: Pinata Cloud
- **Solana Program**: Solana Devnet/Mainnet

---

## Arquitectura del MVP

```
┌─────────────────────────────────────────────────────────────┐
│                     NEXT.JS APP                             │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           CLIENT COMPONENTS (React)                   │  │
│  │  - Upload Page                                        │  │
│  │  - Verify Page                                        │  │
│  │  - Dashboard                                          │  │
│  │  - Certificate View                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓ ↑                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           API ROUTES (/app/api)                       │  │
│  │  - POST /api/certify                                  │  │
│  │  - POST /api/verify                                   │  │
│  │  - GET  /api/certificate/[id]                         │  │
│  │  - GET  /api/search                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓ ↑                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           SERVICES LAYER                              │  │
│  │  - CertificationService                               │  │
│  │  - VerificationService                                │  │
│  │  - SolanaService                                      │  │
│  │  - IPFSService                                        │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          ↓ ↑
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │   Supabase   │  │    Solana    │  │     IPFS     │
    │  PostgreSQL  │  │   Program    │  │   (Pinata)   │
    └──────────────┘  └──────────────┘  └──────────────┘
```

---

## Estructura del Proyecto

```
nft-media-validator/
├── docs/                          # Documentación (ya existe)
├── app/                           # Next.js App Router
│   ├── (routes)/
│   │   ├── page.tsx              # Landing page
│   │   ├── certify/
│   │   │   └── page.tsx          # Página de certificación
│   │   ├── verify/
│   │   │   └── page.tsx          # Página de verificación
│   │   ├── certificate/
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Vista de certificado
│   │   └── dashboard/
│   │       └── page.tsx          # Dashboard de usuario
│   ├── api/
│   │   ├── certify/
│   │   │   └── route.ts          # POST - Certificar medio
│   │   ├── verify/
│   │   │   └── route.ts          # POST - Verificar medio
│   │   ├── certificate/
│   │   │   └── [id]/
│   │   │       └── route.ts      # GET - Obtener certificado
│   │   └── search/
│   │       └── route.ts          # GET - Buscar certificados
│   ├── layout.tsx                # Layout raíz
│   └── globals.css               # Estilos globales
│
├── components/
│   ├── ui/                       # shadcn/ui components
│   ├── UploadZone.tsx           # Drag & drop upload
│   ├── CertificateCard.tsx      # Card de certificado
│   ├── WalletButton.tsx         # Conexión de wallet
│   └── VerificationResult.tsx   # Resultado de verificación
│
├── lib/
│   ├── services/
│   │   ├── certification.ts      # Lógica de certificación
│   │   ├── verification.ts       # Lógica de verificación
│   │   ├── solana.ts            # Interacción con Solana
│   │   ├── ipfs.ts              # Upload/download IPFS
│   │   └── database.ts          # Queries a Supabase
│   ├── utils/
│   │   ├── hash.ts              # Cálculo de hash SHA-256
│   │   ├── watermark.ts         # Aplicación de marca visible
│   │   └── metadata.ts          # Extracción de EXIF
│   └── constants.ts             # Constantes (RPC URLs, etc.)
│
├── solana/                       # Solana Program
│   ├── programs/
│   │   └── media-validator/
│   │       ├── src/
│   │       │   └── lib.rs       # Smart contract en Rust
│   │       └── Cargo.toml
│   ├── tests/
│   │   └── media-validator.ts   # Tests del programa
│   └── Anchor.toml              # Configuración de Anchor
│
├── types/
│   ├── certificate.ts           # Tipos de certificado
│   ├── media.ts                 # Tipos de media
│   └── api.ts                   # Tipos de API
│
├── store/
│   └── useWalletStore.ts        # Zustand store para wallet
│
├── supabase/
│   ├── migrations/
│   │   └── 001_initial_schema.sql
│   └── schema.sql               # Schema completo
│
├── public/
│   ├── images/
│   └── icons/
│
├── .env.local                    # Variables de entorno
├── next.config.js
├── tailwind.config.js
├── tsconfig.json
└── package.json
```

---

## Smart Contract Solana (Rust)

### Program: media_validator

```rust
// solana/programs/media-validator/src/lib.rs

use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod media_validator {
    use super::*;

    pub fn certify_media(
        ctx: Context<CertifyMedia>,
        content_hash: [u8; 32],      // SHA-256 hash
        ipfs_cid: String,             // IPFS CID
        media_type: MediaType,        // Image/Video
        metadata_uri: String,         // URI to JSON metadata
        authenticity_score: u8,       // 0-100
    ) -> Result<()> {
        let certificate = &mut ctx.accounts.certificate;
        let clock = Clock::get()?;

        certificate.content_hash = content_hash;
        certificate.ipfs_cid = ipfs_cid;
        certificate.creator = ctx.accounts.creator.key();
        certificate.timestamp = clock.unix_timestamp;
        certificate.media_type = media_type;
        certificate.metadata_uri = metadata_uri;
        certificate.authenticity_score = authenticity_score;
        certificate.revoked = false;
        certificate.bump = *ctx.bumps.get("certificate").unwrap();

        emit!(MediaCertified {
            content_hash,
            creator: ctx.accounts.creator.key(),
            timestamp: clock.unix_timestamp,
        });

        Ok(())
    }

    pub fn verify_media(
        ctx: Context<VerifyMedia>,
        content_hash: [u8; 32],
    ) -> Result<bool> {
        let certificate = &ctx.accounts.certificate;

        // Verificar que el hash coincida
        require!(
            certificate.content_hash == content_hash,
            ErrorCode::HashMismatch
        );

        // Verificar que no esté revocado
        require!(
            !certificate.revoked,
            ErrorCode::CertificateRevoked
        );

        Ok(true)
    }

    pub fn revoke_certificate(
        ctx: Context<RevokeCertificate>,
        reason: String,
    ) -> Result<()> {
        let certificate = &mut ctx.accounts.certificate;

        require!(
            certificate.creator == ctx.accounts.creator.key(),
            ErrorCode::UnauthorizedRevoke
        );

        certificate.revoked = true;
        certificate.revoke_reason = Some(reason.clone());

        emit!(MediaRevoked {
            content_hash: certificate.content_hash,
            creator: ctx.accounts.creator.key(),
            reason,
        });

        Ok(())
    }
}

// Structs

#[derive(Accounts)]
#[instruction(content_hash: [u8; 32])]
pub struct CertifyMedia<'info> {
    #[account(
        init,
        payer = creator,
        space = Certificate::LEN,
        seeds = [b"certificate", content_hash.as_ref()],
        bump
    )]
    pub certificate: Account<'info, Certificate>,

    #[account(mut)]
    pub creator: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
#[instruction(content_hash: [u8; 32])]
pub struct VerifyMedia<'info> {
    #[account(
        seeds = [b"certificate", content_hash.as_ref()],
        bump = certificate.bump
    )]
    pub certificate: Account<'info, Certificate>,
}

#[derive(Accounts)]
pub struct RevokeCertificate<'info> {
    #[account(
        mut,
        has_one = creator
    )]
    pub certificate: Account<'info, Certificate>,

    pub creator: Signer<'info>,
}

#[account]
pub struct Certificate {
    pub content_hash: [u8; 32],       // 32
    pub ipfs_cid: String,             // 4 + 64 = 68
    pub creator: Pubkey,              // 32
    pub timestamp: i64,               // 8
    pub media_type: MediaType,        // 1
    pub metadata_uri: String,         // 4 + 128 = 132
    pub authenticity_score: u8,       // 1
    pub revoked: bool,                // 1
    pub revoke_reason: Option<String>,// 1 + 4 + 256 = 261
    pub bump: u8,                     // 1
}

impl Certificate {
    pub const LEN: usize = 8 +        // discriminator
                          32 +        // content_hash
                          68 +        // ipfs_cid
                          32 +        // creator
                          8 +         // timestamp
                          1 +         // media_type
                          132 +       // metadata_uri
                          1 +         // authenticity_score
                          1 +         // revoked
                          261 +       // revoke_reason
                          1;          // bump
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum MediaType {
    Image,
    Video,
}

// Events

#[event]
pub struct MediaCertified {
    pub content_hash: [u8; 32],
    pub creator: Pubkey,
    pub timestamp: i64,
}

#[event]
pub struct MediaRevoked {
    pub content_hash: [u8; 32],
    pub creator: Pubkey,
    pub reason: String,
}

// Errors

#[error_code]
pub enum ErrorCode {
    #[msg("Content hash does not match")]
    HashMismatch,
    #[msg("Certificate has been revoked")]
    CertificateRevoked,
    #[msg("Only creator can revoke certificate")]
    UnauthorizedRevoke,
}
```

---

## Schema de Base de Datos (Supabase)

```sql
-- supabase/migrations/001_initial_schema.sql

-- Tabla de certificados (cache de blockchain)
CREATE TABLE certificates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_hash TEXT NOT NULL UNIQUE,
    ipfs_cid TEXT NOT NULL,
    creator_wallet TEXT NOT NULL,
    media_type TEXT NOT NULL CHECK (media_type IN ('image', 'video')),
    file_size BIGINT,
    dimensions JSONB, -- {width, height}
    metadata JSONB,
    authenticity_score INTEGER CHECK (authenticity_score >= 0 AND authenticity_score <= 100),
    revoked BOOLEAN DEFAULT FALSE,
    revoke_reason TEXT,
    solana_signature TEXT NOT NULL,
    solana_account TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Índices
CREATE INDEX idx_certificates_content_hash ON certificates(content_hash);
CREATE INDEX idx_certificates_creator ON certificates(creator_wallet);
CREATE INDEX idx_certificates_created_at ON certificates(created_at DESC);

-- Tabla de verificaciones (analytics)
CREATE TABLE verifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    certificate_id UUID REFERENCES certificates(id),
    status TEXT NOT NULL CHECK (status IN ('verified', 'not_found', 'altered')),
    similarity_score INTEGER,
    verifier_ip TEXT,
    verified_at TIMESTAMPTZ DEFAULT NOW()
);

-- Índices
CREATE INDEX idx_verifications_certificate ON verifications(certificate_id);
CREATE INDEX idx_verifications_verified_at ON verifications(verified_at DESC);

-- Tabla de usuarios (opcional para futuro)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wallet_address TEXT UNIQUE NOT NULL,
    email TEXT,
    reputation_score INTEGER DEFAULT 500,
    total_certifications INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS (Row Level Security)
ALTER TABLE certificates ENABLE ROW LEVEL SECURITY;
ALTER TABLE verifications ENABLE ROW LEVEL SECURITY;

-- Policies (lectura pública, escritura solo autenticado)
CREATE POLICY "Certificates are viewable by everyone"
    ON certificates FOR SELECT
    USING (true);

CREATE POLICY "Verifications are viewable by everyone"
    ON verifications FOR SELECT
    USING (true);
```

---

## Servicios Next.js

### 1. Certification Service

```typescript
// lib/services/certification.ts

import { Connection, PublicKey, Transaction } from '@solana/web3.js';
import { Program, AnchorProvider } from '@coral-xyz/anchor';
import { uploadToIPFS } from './ipfs';
import { calculateHash } from '@/lib/utils/hash';
import { applyWatermark } from '@/lib/utils/watermark';
import { extractMetadata } from '@/lib/utils/metadata';
import { supabase } from './database';

export interface CertificationResult {
  certificateId: string;
  contentHash: string;
  ipfsCid: string;
  solanaTx: string;
  solanaAccount: string;
  authenticityScore: number;
  signedMediaUrl: string;
}

export async function certifyMedia(
  file: File,
  wallet: any,
  connection: Connection,
  program: Program
): Promise<CertificationResult> {

  // 1. Leer archivo
  const buffer = await file.arrayBuffer();
  const uint8Array = new Uint8Array(buffer);

  // 2. Calcular hash original
  const contentHash = calculateHash(uint8Array);

  // 3. Extraer metadata
  const metadata = await extractMetadata(file);

  // 4. Análisis básico de autenticidad (MVP: simplificado)
  const authenticityScore = await analyzeAuthenticity(uint8Array, metadata);

  // 5. Aplicar marca de agua visible
  const watermarkedImage = await applyWatermark(uint8Array, {
    text: `Verified ${new Date().toISOString().split('T')[0]}`,
    position: 'bottom-right',
  });

  // 6. Upload a IPFS
  const ipfsCid = await uploadToIPFS(watermarkedImage);

  // 7. Preparar metadata JSON
  const metadataJson = {
    title: file.name,
    originalFileName: file.name,
    fileSize: file.size,
    mimeType: file.type,
    ...metadata,
    certifiedAt: new Date().toISOString(),
  };

  const metadataIpfsCid = await uploadToIPFS(
    Buffer.from(JSON.stringify(metadataJson))
  );

  // 8. Certificar en Solana
  const contentHashArray = Buffer.from(contentHash, 'hex');

  const [certificatePda] = PublicKey.findProgramAddressSync(
    [Buffer.from('certificate'), contentHashArray],
    program.programId
  );

  const tx = await program.methods
    .certifyMedia(
      Array.from(contentHashArray),
      ipfsCid,
      file.type.startsWith('video') ? { video: {} } : { image: {} },
      metadataIpfsCid,
      authenticityScore
    )
    .accounts({
      certificate: certificatePda,
      creator: wallet.publicKey,
    })
    .rpc();

  // 9. Guardar en Supabase (cache)
  const { data, error } = await supabase
    .from('certificates')
    .insert({
      content_hash: contentHash,
      ipfs_cid: ipfsCid,
      creator_wallet: wallet.publicKey.toString(),
      media_type: file.type.startsWith('video') ? 'video' : 'image',
      file_size: file.size,
      metadata: metadataJson,
      authenticity_score: authenticityScore,
      solana_signature: tx,
      solana_account: certificatePda.toString(),
    })
    .select()
    .single();

  if (error) throw error;

  return {
    certificateId: data.id,
    contentHash,
    ipfsCid,
    solanaTx: tx,
    solanaAccount: certificatePda.toString(),
    authenticityScore,
    signedMediaUrl: `https://ipfs.io/ipfs/${ipfsCid}`,
  };
}

async function analyzeAuthenticity(
  data: Uint8Array,
  metadata: any
): Promise<number> {
  // MVP: Análisis simplificado
  let score = 70; // Base score

  // +10 si tiene metadata EXIF completo
  if (metadata.exif && Object.keys(metadata.exif).length > 5) {
    score += 10;
  }

  // +10 si tiene timestamp razonable
  if (metadata.timestamp) {
    score += 10;
  }

  // +10 si tiene información de cámara
  if (metadata.camera) {
    score += 10;
  }

  return Math.min(score, 100);
}
```

### 2. Verification Service

```typescript
// lib/services/verification.ts

import { Connection, PublicKey } from '@solana/web3.js';
import { Program } from '@coral-xyz/anchor';
import { calculateHash } from '@/lib/utils/hash';
import { supabase } from './database';

export interface VerificationResult {
  status: 'verified' | 'not_found' | 'altered';
  exists: boolean;
  certificate?: {
    contentHash: string;
    creator: string;
    certifiedAt: string;
    ipfsCid: string;
    authenticityScore: number;
  };
  similarity?: number;
}

export async function verifyMedia(
  file: File,
  connection: Connection,
  program: Program
): Promise<VerificationResult> {

  // 1. Calcular hash del archivo
  const buffer = await file.arrayBuffer();
  const contentHash = calculateHash(new Uint8Array(buffer));

  // 2. Buscar en cache (Supabase)
  const { data: cachedCert } = await supabase
    .from('certificates')
    .select('*')
    .eq('content_hash', contentHash)
    .single();

  if (cachedCert) {
    // Registrar verificación
    await supabase.from('verifications').insert({
      certificate_id: cachedCert.id,
      status: 'verified',
      similarity_score: 100,
    });

    return {
      status: 'verified',
      exists: true,
      certificate: {
        contentHash: cachedCert.content_hash,
        creator: cachedCert.creator_wallet,
        certifiedAt: cachedCert.created_at,
        ipfsCid: cachedCert.ipfs_cid,
        authenticityScore: cachedCert.authenticity_score,
      },
      similarity: 100,
    };
  }

  // 3. Verificar en blockchain directamente
  const contentHashArray = Buffer.from(contentHash, 'hex');

  const [certificatePda] = PublicKey.findProgramAddressSync(
    [Buffer.from('certificate'), contentHashArray],
    program.programId
  );

  try {
    const certificateAccount = await program.account.certificate.fetch(
      certificatePda
    );

    // Encontrado en blockchain pero no en cache
    // Sincronizar a Supabase
    await syncCertificateToDatabase(certificateAccount, certificatePda);

    return {
      status: 'verified',
      exists: true,
      certificate: {
        contentHash,
        creator: certificateAccount.creator.toString(),
        certifiedAt: new Date(certificateAccount.timestamp * 1000).toISOString(),
        ipfsCid: certificateAccount.ipfsCid,
        authenticityScore: certificateAccount.authenticityScore,
      },
      similarity: 100,
    };
  } catch (error) {
    // No encontrado
    return {
      status: 'not_found',
      exists: false,
    };
  }
}

async function syncCertificateToDatabase(cert: any, pda: PublicKey) {
  await supabase.from('certificates').insert({
    content_hash: Buffer.from(cert.contentHash).toString('hex'),
    ipfs_cid: cert.ipfsCid,
    creator_wallet: cert.creator.toString(),
    media_type: cert.mediaType.image ? 'image' : 'video',
    metadata: { uri: cert.metadataUri },
    authenticity_score: cert.authenticityScore,
    revoked: cert.revoked,
    solana_account: pda.toString(),
  });
}
```

---

## API Routes (Next.js)

### POST /api/certify

```typescript
// app/api/certify/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { Connection } from '@solana/web3.js';
import { AnchorProvider, Program } from '@coral-xyz/anchor';
import { certifyMedia } from '@/lib/services/certification';
import idl from '@/solana/target/idl/media_validator.json';

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;
    const walletPublicKey = formData.get('walletPublicKey') as string;

    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }

    // Setup Solana connection
    const connection = new Connection(
      process.env.NEXT_PUBLIC_SOLANA_RPC_URL!,
      'confirmed'
    );

    // Setup program
    // Note: En MVP el wallet firma del lado del cliente
    // Aquí solo procesamos el resultado

    const result = await certifyMedia(
      file,
      { publicKey: walletPublicKey },
      connection,
      program
    );

    return NextResponse.json(result);
  } catch (error: any) {
    console.error('Certification error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}
```

### POST /api/verify

```typescript
// app/api/verify/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { Connection } from '@solana/web3.js';
import { Program } from '@coral-xyz/anchor';
import { verifyMedia } from '@/lib/services/verification';

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;

    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }

    const connection = new Connection(
      process.env.NEXT_PUBLIC_SOLANA_RPC_URL!,
      'confirmed'
    );

    // Setup program
    const program = getProgram(connection);

    const result = await verifyMedia(file, connection, program);

    return NextResponse.json(result);
  } catch (error: any) {
    console.error('Verification error:', error);
    return NextResponse.json(
      { error: error.message },
      { status: 500 }
    );
  }
}
```

---

## Componentes Frontend

### Upload Zone

```typescript
// components/UploadZone.tsx

'use client';

import { useCallback, useState } from 'react';
import { useDropzone } from 'react-dropzone';
import { Upload } from 'lucide-react';

interface UploadZoneProps {
  onFileSelect: (file: File) => void;
}

export function UploadZone({ onFileSelect }: UploadZoneProps) {
  const [preview, setPreview] = useState<string | null>(null);

  const onDrop = useCallback((acceptedFiles: File[]) => {
    const file = acceptedFiles[0];
    if (file) {
      onFileSelect(file);

      // Generate preview
      const reader = new FileReader();
      reader.onload = () => {
        setPreview(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  }, [onFileSelect]);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'image/*': ['.png', '.jpg', '.jpeg'],
      'video/*': ['.mp4', '.mov'],
    },
    maxFiles: 1,
    maxSize: 100 * 1024 * 1024, // 100MB
  });

  return (
    <div
      {...getRootProps()}
      className={`
        border-2 border-dashed rounded-lg p-12 text-center cursor-pointer
        transition-colors
        ${isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300'}
      `}
    >
      <input {...getInputProps()} />

      {preview ? (
        <div className="space-y-4">
          {preview.startsWith('data:image') ? (
            <img src={preview} alt="Preview" className="max-h-64 mx-auto" />
          ) : (
            <video src={preview} className="max-h-64 mx-auto" controls />
          )}
          <p className="text-sm text-gray-600">
            Click to change or drag another file
          </p>
        </div>
      ) : (
        <div className="space-y-4">
          <Upload className="w-12 h-12 mx-auto text-gray-400" />
          <div>
            <p className="text-lg font-medium">
              Drop your image or video here
            </p>
            <p className="text-sm text-gray-500">
              or click to browse
            </p>
          </div>
          <p className="text-xs text-gray-400">
            Supports: JPEG, PNG, MP4, MOV (max 100MB)
          </p>
        </div>
      )}
    </div>
  );
}
```

### Certification Page

```typescript
// app/certify/page.tsx

'use client';

import { useState } from 'react';
import { useWallet } from '@solana/wallet-adapter-react';
import { UploadZone } from '@/components/UploadZone';
import { Button } from '@/components/ui/button';
import { WalletButton } from '@/components/WalletButton';

export default function CertifyPage() {
  const { publicKey, signTransaction } = useWallet();
  const [file, setFile] = useState<File | null>(null);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<any>(null);

  const handleCertify = async () => {
    if (!file || !publicKey) return;

    setLoading(true);

    try {
      const formData = new FormData();
      formData.append('file', file);
      formData.append('walletPublicKey', publicKey.toString());

      const response = await fetch('/api/certify', {
        method: 'POST',
        body: formData,
      });

      const data = await response.json();
      setResult(data);
    } catch (error) {
      console.error('Certification failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="container max-w-4xl mx-auto py-12 px-4">
      <div className="space-y-8">
        <div>
          <h1 className="text-4xl font-bold">Certify Media</h1>
          <p className="text-gray-600 mt-2">
            Upload your image or video to certify its authenticity
          </p>
        </div>

        {!publicKey ? (
          <div className="text-center py-12">
            <p className="mb-4">Connect your wallet to certify media</p>
            <WalletButton />
          </div>
        ) : (
          <>
            <UploadZone onFileSelect={setFile} />

            {file && (
              <div className="flex gap-4">
                <Button
                  onClick={handleCertify}
                  disabled={loading}
                  size="lg"
                  className="w-full"
                >
                  {loading ? 'Certifying...' : 'Certify Media'}
                </Button>
              </div>
            )}

            {result && (
              <div className="border rounded-lg p-6 bg-green-50">
                <h3 className="font-bold text-green-800 mb-4">
                  ✓ Media Certified Successfully!
                </h3>
                <dl className="space-y-2 text-sm">
                  <div>
                    <dt className="font-medium">Certificate ID:</dt>
                    <dd className="font-mono text-xs">{result.certificateId}</dd>
                  </div>
                  <div>
                    <dt className="font-medium">Content Hash:</dt>
                    <dd className="font-mono text-xs">{result.contentHash}</dd>
                  </div>
                  <div>
                    <dt className="font-medium">Solana Transaction:</dt>
                    <dd>
                      <a
                        href={`https://explorer.solana.com/tx/${result.solanaTx}?cluster=devnet`}
                        target="_blank"
                        className="text-blue-600 hover:underline text-xs"
                      >
                        View on Solana Explorer
                      </a>
                    </dd>
                  </div>
                  <div>
                    <dt className="font-medium">Authenticity Score:</dt>
                    <dd>{result.authenticityScore}/100</dd>
                  </div>
                </dl>
              </div>
            )}
          </>
        )}
      </div>
    </div>
  );
}
```

---

## Variables de Entorno

```bash
# .env.local

# Solana
NEXT_PUBLIC_SOLANA_NETWORK=devnet
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.devnet.solana.com
# Para producción usar Helius o QuickNode:
# NEXT_PUBLIC_SOLANA_RPC_URL=https://mainnet.helius-rpc.com/?api-key=your-key

# Solana Program ID (después de deploy)
NEXT_PUBLIC_PROGRAM_ID=your_program_id_here

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# IPFS (Pinata)
PINATA_API_KEY=your_pinata_api_key
PINATA_SECRET_KEY=your_pinata_secret_key
PINATA_JWT=your_pinata_jwt

# Redis (Upstash) - opcional para cache
UPSTASH_REDIS_URL=your_upstash_url
UPSTASH_REDIS_TOKEN=your_upstash_token

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## Plan de Desarrollo (8 Semanas)

### Semana 1: Setup & Smart Contract

**Días 1-2**: Configuración del proyecto
- [ ] Inicializar Next.js project
- [ ] Setup TailwindCSS + shadcn/ui
- [ ] Configurar TypeScript
- [ ] Setup git repository

**Días 3-5**: Solana Program
- [ ] Inicializar Anchor project
- [ ] Escribir smart contract básico
- [ ] Tests unitarios
- [ ] Deploy a Devnet

**Días 6-7**: Supabase Setup
- [ ] Crear proyecto Supabase
- [ ] Ejecutar migrations
- [ ] Configurar RLS
- [ ] Test conexión desde Next.js

---

### Semana 2: Backend Services

**Días 1-3**: Servicios Core
- [ ] IPFSService (upload/download)
- [ ] Hash utilities
- [ ] Metadata extraction (EXIF)
- [ ] Database helpers

**Días 4-5**: Certification Service
- [ ] Implementar `certifyMedia()`
- [ ] Integración con Solana
- [ ] Tests

**Días 6-7**: Verification Service
- [ ] Implementar `verifyMedia()`
- [ ] Query a blockchain
- [ ] Tests

---

### Semana 3: API Routes

**Días 1-2**: API Endpoints
- [ ] POST /api/certify
- [ ] POST /api/verify
- [ ] GET /api/certificate/[id]

**Días 3-4**: Error Handling & Validation
- [ ] Zod schemas
- [ ] Error responses
- [ ] Rate limiting básico

**Días 5-7**: Testing
- [ ] Test de integración de APIs
- [ ] Postman collection
- [ ] Documentación de endpoints

---

### Semana 4: Frontend Core

**Días 1-2**: Layout & Navigation
- [ ] Layout principal
- [ ] Navigation bar
- [ ] Wallet integration (Solana Wallet Adapter)

**Días 3-4**: Components
- [ ] UploadZone
- [ ] WalletButton
- [ ] CertificateCard
- [ ] VerificationResult

**Días 5-7**: Pages
- [ ] Landing page
- [ ] Certify page
- [ ] Verify page

---

### Semana 5: Features Avanzadas

**Días 1-3**: Certificate View
- [ ] Página de certificado individual
- [ ] Metadata display
- [ ] QR code generation
- [ ] Share functionality

**Días 4-5**: Dashboard
- [ ] Lista de certificados del usuario
- [ ] Filtros y búsqueda
- [ ] Estadísticas básicas

**Días 6-7**: Watermarking
- [ ] Implementar marca de agua visible
- [ ] Configuración de posición
- [ ] Preview antes de certificar

---

### Semana 6: Polish & UX

**Días 1-2**: UI/UX Improvements
- [ ] Loading states
- [ ] Empty states
- [ ] Error states
- [ ] Animations (Framer Motion)

**Días 3-4**: Responsive Design
- [ ] Mobile optimization
- [ ] Tablet views
- [ ] Desktop enhancements

**Días 5-7**: Performance
- [ ] Image optimization (next/image)
- [ ] Code splitting
- [ ] Lazy loading
- [ ] SEO basics

---

### Semana 7: Testing & Refinement

**Días 1-3**: Testing
- [ ] E2E tests (Playwright)
- [ ] Unit tests (Jest)
- [ ] Integration tests
- [ ] Bug fixes

**Días 4-5**: Security
- [ ] Input sanitization
- [ ] CSRF protection
- [ ] Rate limiting robusto
- [ ] Security audit

**Días 6-7**: Documentation
- [ ] README actualizado
- [ ] API documentation
- [ ] User guide
- [ ] Developer guide

---

### Semana 8: Deploy & Launch

**Días 1-2**: Pre-Deploy
- [ ] Environment variables en Vercel
- [ ] Build tests
- [ ] Performance checks
- [ ] Lighthouse audit

**Días 3-4**: Deploy
- [ ] Deploy Solana program a Devnet
- [ ] Deploy Next.js a Vercel
- [ ] Configurar dominio
- [ ] SSL certificates

**Días 5-6**: Testing en Producción
- [ ] Smoke tests
- [ ] User acceptance testing
- [ ] Bug fixes

**Día 7**: Launch
- [ ] Documentación final
- [ ] Anuncio en redes
- [ ] Monitoring setup
- [ ] 🎉 Launch!

---

## Costos Estimados del MVP

### Desarrollo
- **Personal** (1 full-stack developer, 8 semanas): $16,000
- **Total Desarrollo**: $16,000

### Infraestructura (mensual)
- **Vercel** (Pro plan): $20/mes
- **Supabase** (Pro plan): $25/mes
- **Pinata** (Submarine plan): $20/mes
- **Helius RPC** (Developer): $49/mes
- **Dominio**: $12/año
- **Total Mensual**: ~$115/mes

### One-time
- **Solana Program Deploy**: ~1 SOL (~$100)
- **Design/Branding**: $500
- **Total One-time**: ~$600

### Total MVP: $16,600 + $115/mes

---

## Métricas de Éxito del MVP

### Técnicas
- [ ] Tiempo de certificación < 10 segundos
- [ ] Tiempo de verificación < 2 segundos
- [ ] Uptime > 99%
- [ ] 0 vulnerabilidades críticas

### Negocio
- [ ] 50 usuarios registrados (wallets conectadas)
- [ ] 100 certificaciones creadas
- [ ] 500 verificaciones realizadas
- [ ] 10 usuarios activos semanalmente

### Feedback
- [ ] 5 entrevistas con usuarios
- [ ] NPS > 40
- [ ] < 3 bugs críticos reportados

---

## Próximos Pasos Post-MVP

1. **Esteganografía invisible** (sello invisible en imágenes)
2. **Detección de IA** (modelo ML básico)
3. **Soporte para videos** (completo con watermarking)
4. **Mobile app** (React Native)
5. **Analytics dashboard** (métricas de uso)
6. **API pública** (para terceros)
7. **Mainnet deployment** (migrar de Devnet)

---

**MVP enfocado en demostrar valor core: Certificación verificable en blockchain con UX simple.**
