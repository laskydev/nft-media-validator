# Implementación Blockchain

## Arquitectura Descentralizada para Registro Inmutable

### Selección de Blockchain

#### Evaluación de Opciones

| Blockchain | Pros | Contras | Score |
|------------|------|---------|-------|
| **Ethereum** | Máxima seguridad, ecosistema maduro | Gas fees altos, throughput limitado | 7/10 |
| **Polygon** | Compatible EVM, bajos costos, buena velocidad | Menor descentralización que Ethereum | 9/10 |
| **Solana** | Muy rápido, costos bajísimos | Outages históricos, menos maduro | 6/10 |
| **Avalanche** | Subnets personalizadas, rápido | Ecosistema más pequeño | 7/10 |
| **Arbitrum** | L2 de Ethereum, económico | Complejidad adicional | 8/10 |

#### Decisión: Polygon (Fase 1) + Multi-Chain (Fase 2)

**Justificación para Polygon**:
1. **Costo**: ~$0.001 por transacción (vs $5-50 en Ethereum)
2. **Velocidad**: ~2 segundos por bloque
3. **Compatibilidad**: EVM-compatible (fácil migración a/desde Ethereum)
4. **Adopción**: Usado por marcas reconocidas (Instagram, Reddit)
5. **Seguridad**: Hereda seguridad de Ethereum vía checkpoints

**Roadmap Multi-Chain**:
- Fase 1: Polygon (2025 Q1-Q2)
- Fase 2: Ethereum mainnet para contenido crítico (2025 Q3)
- Fase 3: Avalanche para medios de alto volumen (2025 Q4)
- Fase 4: Bridge entre cadenas (2026)

### Arquitectura de Smart Contracts

#### Contrato Principal: MediaCertificate.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

/**
 * @title MediaCertificate
 * @dev Registro inmutable de certificaciones de medios
 */
contract MediaCertificate is Ownable, Pausable, ReentrancyGuard {

    struct Certificate {
        bytes32 contentHash;        // SHA-256 del contenido
        string ipfsCID;             // IPFS Content ID
        address creator;            // Wallet del creador
        uint256 timestamp;          // Timestamp de certificación
        MediaType mediaType;        // Tipo de medio
        string metadataURI;         // URI a metadata (IPFS)
        uint8 authenticityScore;    // Score 0-100
        bool revoked;               // Flag de revocación
        string revokeReason;        // Razón de revocación
    }

    enum MediaType {
        IMAGE,
        VIDEO,
        AUDIO
    }

    // Mappings
    mapping(bytes32 => Certificate) public certificates;
    mapping(address => bytes32[]) public creatorCertificates;
    mapping(bytes32 => Certificate[]) public certificateHistory;

    // Contadores
    uint256 public totalCertifications;
    uint256 public totalCreators;

    // Eventos
    event MediaCertified(
        bytes32 indexed contentHash,
        address indexed creator,
        string ipfsCID,
        uint256 timestamp,
        uint8 authenticityScore
    );

    event MediaRevoked(
        bytes32 indexed contentHash,
        address indexed revoker,
        string reason,
        uint256 timestamp
    );

    event MediaUpdated(
        bytes32 indexed oldHash,
        bytes32 indexed newHash,
        address indexed creator,
        string changeDescription
    );

    /**
     * @dev Certifica un nuevo medio
     */
    function certifyMedia(
        bytes32 _contentHash,
        string memory _ipfsCID,
        MediaType _mediaType,
        string memory _metadataURI,
        uint8 _authenticityScore
    ) external whenNotPaused nonReentrant returns (bool) {
        require(_contentHash != bytes32(0), "Invalid content hash");
        require(certificates[_contentHash].timestamp == 0, "Already certified");
        require(_authenticityScore <= 100, "Invalid score");

        Certificate memory cert = Certificate({
            contentHash: _contentHash,
            ipfsCID: _ipfsCID,
            creator: msg.sender,
            timestamp: block.timestamp,
            mediaType: _mediaType,
            metadataURI: _metadataURI,
            authenticityScore: _authenticityScore,
            revoked: false,
            revokeReason: ""
        });

        certificates[_contentHash] = cert;
        creatorCertificates[msg.sender].push(_contentHash);
        certificateHistory[_contentHash].push(cert);

        if (creatorCertificates[msg.sender].length == 1) {
            totalCreators++;
        }
        totalCertifications++;

        emit MediaCertified(
            _contentHash,
            msg.sender,
            _ipfsCID,
            block.timestamp,
            _authenticityScore
        );

        return true;
    }

    /**
     * @dev Revoca una certificación (solo el creador)
     */
    function revokeMedia(
        bytes32 _contentHash,
        string memory _reason
    ) external {
        Certificate storage cert = certificates[_contentHash];
        require(cert.timestamp != 0, "Certificate not found");
        require(cert.creator == msg.sender, "Not the creator");
        require(!cert.revoked, "Already revoked");

        cert.revoked = true;
        cert.revokeReason = _reason;

        emit MediaRevoked(_contentHash, msg.sender, _reason, block.timestamp);
    }

    /**
     * @dev Registra una nueva versión de un medio editado
     */
    function updateMedia(
        bytes32 _oldHash,
        bytes32 _newHash,
        string memory _newIpfsCID,
        string memory _newMetadataURI,
        string memory _changeDescription,
        uint8 _newAuthenticityScore
    ) external whenNotPaused nonReentrant returns (bool) {
        Certificate storage oldCert = certificates[_oldHash];
        require(oldCert.timestamp != 0, "Original not found");
        require(oldCert.creator == msg.sender, "Not the creator");
        require(certificates[_newHash].timestamp == 0, "New hash already exists");

        Certificate memory newCert = Certificate({
            contentHash: _newHash,
            ipfsCID: _newIpfsCID,
            creator: msg.sender,
            timestamp: block.timestamp,
            mediaType: oldCert.mediaType,
            metadataURI: _newMetadataURI,
            authenticityScore: _newAuthenticityScore,
            revoked: false,
            revokeReason: ""
        });

        certificates[_newHash] = newCert;
        creatorCertificates[msg.sender].push(_newHash);
        certificateHistory[_oldHash].push(newCert);
        certificateHistory[_newHash].push(newCert);

        totalCertifications++;

        emit MediaUpdated(_oldHash, _newHash, msg.sender, _changeDescription);
        emit MediaCertified(
            _newHash,
            msg.sender,
            _newIpfsCID,
            block.timestamp,
            _newAuthenticityScore
        );

        return true;
    }

    /**
     * @dev Obtiene certificación por hash
     */
    function getCertificate(bytes32 _contentHash)
        external
        view
        returns (Certificate memory)
    {
        return certificates[_contentHash];
    }

    /**
     * @dev Obtiene todas las certificaciones de un creador
     */
    function getCreatorCertificates(address _creator)
        external
        view
        returns (bytes32[] memory)
    {
        return creatorCertificates[_creator];
    }

    /**
     * @dev Obtiene historial de versiones de un medio
     */
    function getCertificateHistory(bytes32 _contentHash)
        external
        view
        returns (Certificate[] memory)
    {
        return certificateHistory[_contentHash];
    }

    /**
     * @dev Verifica si un medio está certificado y válido
     */
    function isVerified(bytes32 _contentHash)
        external
        view
        returns (bool)
    {
        Certificate memory cert = certificates[_contentHash];
        return cert.timestamp != 0 && !cert.revoked;
    }

    /**
     * @dev Pausar contrato en caso de emergencia
     */
    function pause() external onlyOwner {
        _pause();
    }

    /**
     * @dev Reanudar contrato
     */
    function unpause() external onlyOwner {
        _unpause();
    }
}
```

#### Contrato de Reputación: CreatorReputation.sol

```solidity
/**
 * @title CreatorReputation
 * @dev Sistema de reputación para creadores
 */
contract CreatorReputation is Ownable {

    struct Reputation {
        uint256 score;              // Score 0-1000
        uint256 totalCertifications;
        uint256 totalVerifications;
        uint256 validReports;       // Reportes que fueron válidos
        uint256 invalidReports;     // Reportes desestimados
        uint256 memberSince;
    }

    mapping(address => Reputation) public reputations;

    event ReputationUpdated(
        address indexed creator,
        uint256 newScore,
        string reason
    );

    /**
     * @dev Incrementa reputación por certificación exitosa
     */
    function incrementCertification(address _creator) external onlyOwner {
        if (reputations[_creator].memberSince == 0) {
            reputations[_creator].memberSince = block.timestamp;
            reputations[_creator].score = 500; // Start at 500
        }

        reputations[_creator].totalCertifications++;

        // +1 point per certification, max 800 from certifications alone
        if (reputations[_creator].score < 800) {
            reputations[_creator].score++;
        }

        emit ReputationUpdated(_creator, reputations[_creator].score, "Certification");
    }

    /**
     * @dev Penaliza por reporte válido
     */
    function penalizeForReport(address _creator, uint256 _severity) external onlyOwner {
        reputations[_creator].validReports++;

        // Severity: 1 (minor) to 100 (severe)
        if (reputations[_creator].score > _severity) {
            reputations[_creator].score -= _severity;
        } else {
            reputations[_creator].score = 0;
        }

        emit ReputationUpdated(_creator, reputations[_creator].score, "Penalty");
    }

    /**
     * @dev Obtiene score de reputación
     */
    function getReputation(address _creator) external view returns (uint256) {
        return reputations[_creator].score;
    }
}
```

### Optimizaciones de Gas

#### Estrategias Implementadas

**1. Batch Certification**
```solidity
function certifyBatch(
    bytes32[] memory _hashes,
    string[] memory _ipfsCIDs,
    MediaType[] memory _types,
    string memory _batchMetadataURI,
    uint8[] memory _scores
) external whenNotPaused nonReentrant returns (bool) {
    require(_hashes.length == _ipfsCIDs.length, "Length mismatch");
    require(_hashes.length <= 100, "Batch too large");

    for (uint i = 0; i < _hashes.length; i++) {
        // Certificación individual sin eventos individuales
        _certifyMediaInternal(_hashes[i], _ipfsCIDs[i], _types[i], "", _scores[i]);
    }

    // Un solo evento para todo el batch
    emit BatchCertified(_hashes.length, msg.sender, block.timestamp);

    return true;
}
```

**2. Metadata Off-Chain**
- Almacenar metadata extensa en IPFS
- Solo hash/CID en blockchain
- Ahorro: ~70% de gas vs metadata on-chain

**3. Struct Packing**
```solidity
struct Certificate {
    bytes32 contentHash;        // 32 bytes
    address creator;            // 20 bytes
    uint128 timestamp;          // 16 bytes (suficiente hasta año 10^28)
    uint8 authenticityScore;    // 1 byte
    bool revoked;               // 1 byte
    // Total: 70 bytes vs 96+ bytes sin optimizar
}
```

**4. Eventos en lugar de Storage**
- Información histórica en eventos (indexable off-chain)
- Solo estado actual en storage
- Ahorro: ~80% para datos históricos

### Seguridad del Smart Contract

#### Auditorías Programadas

**Pre-Launch**:
- [ ] CertiK Audit
- [ ] OpenZeppelin Review
- [ ] Internal Security Review
- [ ] Community Bug Bounty (testnet)

**Post-Launch**:
- [ ] Trimestral: Code review de updates
- [ ] Anual: Full re-audit
- [ ] Continuo: Monitoring on-chain activity

#### Mejores Prácticas Implementadas

```
✓ OpenZeppelin contracts base
✓ ReentrancyGuard en funciones críticas
✓ Pausable para emergencias
✓ Access control (Ownable, roles)
✓ Input validation exhaustiva
✓ Eventos comprehensivos
✓ SafeMath (implícito en Solidity 0.8+)
✓ Checks-Effects-Interactions pattern
✓ No delegatecall a contratos no confiables
✓ Límites en loops (max 100 items)
```

### Interacción Off-Chain ↔ On-Chain

#### Flujo de Certificación

```
1. Backend:
   - Procesa medio
   - Calcula hash
   - Sube a IPFS → obtiene CID
   - Prepara metadata → sube a IPFS → obtiene CID

2. Backend (Blockchain Service):
   - Firma transacción con wallet del sistema
   - Llama a certifyMedia()
   - Espera confirmación (12 bloques en Polygon)

3. Blockchain:
   - Ejecuta smart contract
   - Almacena certificación
   - Emite evento MediaCertified

4. Backend (Event Listener):
   - Escucha evento
   - Actualiza database PostgreSQL (cache)
   - Retorna confirmación a cliente
```

#### Infraestructura de Nodos

**Fase 1: Nodos Gestionados**
- Alchemy/Infura para Polygon
- RPC endpoints redundantes
- Fallback automático si falla primario

**Fase 2: Nodos Propios**
- Nodo completo de Polygon (archivo)
- Nodo de Ethereum (L1) para checkpoints
- Sincronización continua
- Beneficio: No dependencia de terceros

### Gas Economics

#### Estimación de Costos

```
Operación              Gas Estimado    Costo (MATIC)   Costo (USD)*
─────────────────────────────────────────────────────────────────────
Certificación simple   ~100,000        ~0.001          ~$0.0008
Certificación batch    ~50,000/item    ~0.0005/item    ~$0.0004/item
Revocación            ~30,000         ~0.0003         ~$0.00024
Update (nueva versión) ~120,000        ~0.0012         ~$0.00096
Query (read-only)      0               0               $0

* Precio MATIC = $0.80, Gas price = 100 gwei
```

#### Subsidios de Gas (Opcional)

**Meta-Transactions**:
- Usuarios firman intención (off-chain)
- Relayer paga gas y ejecuta transacción
- Usuario paga en fiat/stablecoins
- Tecnología: Biconomy, OpenZeppelin Defender

**Ventajas**:
- Onboarding sin fricción (no need MATIC)
- UX tradicional Web2
- Mayor adopción

### Indexación y Queries

#### The Graph Protocol

```graphql
# Schema.graphql
type Certificate @entity {
  id: ID!                           # contentHash
  contentHash: Bytes!
  ipfsCID: String!
  creator: Creator!
  timestamp: BigInt!
  mediaType: MediaType!
  metadataURI: String!
  authenticityScore: Int!
  revoked: Boolean!
  revokeReason: String
  history: [Certificate!]!
}

type Creator @entity {
  id: ID!                           # address
  address: Bytes!
  certificates: [Certificate!]! @derivedFrom(field: "creator")
  totalCertifications: BigInt!
  reputation: BigInt!
  memberSince: BigInt!
}

enum MediaType {
  IMAGE
  VIDEO
  AUDIO
}
```

**Subgraph Deployment**:
- Indexa todos los eventos del smart contract
- Query GraphQL performante
- Subscriptions en tiempo real

**Ejemplo Query**:
```graphql
{
  certificates(
    where: { creator: "0x742d...", revoked: false }
    orderBy: timestamp
    orderDirection: desc
    first: 10
  ) {
    contentHash
    ipfsCID
    timestamp
    authenticityScore
  }
}
```

### Multi-Chain Strategy

#### Bridge entre Chains

**Escenario**: Certificado en Polygon, verificación desde Ethereum

```
Solución:
1. Cross-chain message passing (Hyperlane, LayerZero)
2. Light client verification
3. Merkle proofs de estado

Implementación:
- Contract en Ethereum: Verifier.sol
- Contract en Polygon: MediaCertificate.sol
- Bridge contract: MediaCertificateBridge.sol

function verifyCrossChain(
    bytes32 _contentHash,
    bytes memory _proof
) external view returns (bool) {
    // Verifica Merkle proof de estado de Polygon
    // Sin necesidad de replicar todos los datos
}
```

### Gobernanza Descentralizada (Futuro)

#### DAO para Decisiones Críticas

**Fase 3: Transición a DAO**

```solidity
contract MediaValidatorDAO {
    // Token de gobernanza
    IERC20 public governanceToken;

    // Propuestas
    struct Proposal {
        uint256 id;
        address proposer;
        string description;
        uint256 votesFor;
        uint256 votesAgainst;
        uint256 endTime;
        bool executed;
    }

    // Tipos de decisiones votables:
    // - Actualización de smart contracts
    // - Parámetros del sistema (fees, scores)
    // - Incorporación de nuevas blockchains
    // - Resolución de disputas complejas
}
```

**Distribución de Tokens**:
- 40% Creadores (según reputación y uso)
- 30% Early adopters y contribuidores
- 20% Treasury para desarrollo
- 10% Team

---

**Blockchain: La fuente de verdad que nadie puede silenciar**
