# OPTYMA AGRITECH PROTOCOL - Backend & AI Architecture

## 🔧 Arsitektur Backend

### Tech Stack Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Backend Architecture                     │
├─────────────────────────────────────────────────────────────┤
│  Node.js + Express.js + TypeScript + PostgreSQL + Redis    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │     API     │  │  Database   │  │   AI/ML     │         │
│  │  Gateway    │  │   Layer     │  │  Services   │         │
│  │             │  │             │  │             │         │
│  │ • Express   │  │ • PostgreSQL│  │ • Python    │         │
│  │ • GraphQL   │  │ • Prisma    │  │ • FastAPI   │         │
│  │ • Auth      │  │ • Redis     │  │ • TensorFlow│         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Project Structure

```
optyma-backend/
├── src/
│   ├── api/                       # API routes
│   │   ├── auth/
│   │   ├── agridaos/
│   │   ├── consultants/
│   │   ├── knowledge/
│   │   └── blockchain/
│   ├── services/                  # Business logic
│   │   ├── AuthService.ts
│   │   ├── BlockchainService.ts
│   │   ├── AIService.ts
│   │   └── NotificationService.ts
│   ├── models/                    # Database models
│   │   ├── User.ts
│   │   ├── AgriDAO.ts
│   │   ├── Consultant.ts
│   │   └── KnowledgeToken.ts
│   ├── middleware/                # Express middleware
│   │   ├── auth.ts
│   │   ├── validation.ts
│   │   └── rateLimit.ts
│   ├── utils/                     # Utilities
│   │   ├── encryption.ts
│   │   ├── ipfs.ts
│   │   └── websocket.ts
│   ├── config/                    # Configuration
│   │   ├── database.ts
│   │   ├── redis.ts
│   │   └── blockchain.ts
│   └── app.ts                     # Express app setup
├── ai-services/                   # Python AI services
│   ├── agents/
│   │   ├── consultant_agent.py
│   │   ├── diagnosis_agent.py
│   │   └── market_agent.py
│   ├── models/
│   │   ├── nlp_model.py
│   │   ├── image_classifier.py
│   │   └── price_predictor.py
│   ├── data/
│   │   ├── training_data/
│   │   └── knowledge_base/
│   └── main.py                    # FastAPI app
├── prisma/                        # Database schema
│   ├── schema.prisma
│   └── migrations/
├── docker-compose.yml
└── package.json
```

## 🗄️ Database Schema

### PostgreSQL Database Design

```sql
-- prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id              String    @id @default(cuid())
  walletAddress   String    @unique
  email           String?   @unique
  name            String?
  avatar          String?
  isConsultant    Boolean   @default(false)
  isVerified      Boolean   @default(false)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  // Relations
  consultantProfile ConsultantProfile?
  agriDAOs          AgriDAOMember[]
  knowledgeTokens   KnowledgeToken[]
  consultations     Consultation[]
  reviews           Review[]

  @@map("users")
}

model ConsultantProfile {
  id                String   @id @default(cuid())
  userId            String   @unique
  expertise         String   // "aquaculture", "livestock", "crops"
  yearsExperience   Int
  certifications    String[] // Array of certification URLs
  biography         String?
  consultationFee   Decimal  @db.Decimal(10, 4)
  isAvailable       Boolean  @default(true)
  rating            Decimal? @db.Decimal(3, 2)
  totalConsultations Int     @default(0)
  
  // Relations
  user              User            @relation(fields: [userId], references: [id])
  consultations     Consultation[]
  knowledgeTokens   KnowledgeToken[]

  @@map("consultant_profiles")
}

model AgriDAO {
  id              String    @id @default(cuid())
  contractAddress String    @unique
  name            String
  symbol          String
  specialty       String    // "aquaculture", "livestock", etc.
  description     String?
  targetRaise     Decimal   @db.Decimal(18, 4)
  currentRaise    Decimal   @db.Decimal(18, 4) @default(0)
  isActive        Boolean   @default(true)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  // Relations
  members         AgriDAOMember[]
  projects        Project[]
  consultations   Consultation[]
  knowledgeTokens KnowledgeToken[]

  @@map("agri_daos")
}

model AgriDAOMember {
  id        String @id @default(cuid())
  userId    String
  daoId     String
  role      String @default("member") // "member", "consultant", "admin"
  tokensOwned Decimal @db.Decimal(18, 4) @default(0)
  joinedAt  DateTime @default(now())

  // Relations
  user      User    @relation(fields: [userId], references: [id])
  dao       AgriDAO @relation(fields: [daoId], references: [id])

  @@unique([userId, daoId])
  @@map("agri_dao_members")
}

model Project {
  id            String   @id @default(cuid())
  daoId         String
  title         String
  description   String
  fundingGoal   Decimal  @db.Decimal(18, 4)
  currentFunding Decimal @db.Decimal(18, 4) @default(0)
  proposerId    String
  isActive      Boolean  @default(true)
  deadline      DateTime
  createdAt     DateTime @default(now())

  // Relations
  dao           AgriDAO  @relation(fields: [daoId], references: [id])
  fundings      ProjectFunding[]

  @@map("projects")
}

model ProjectFunding {
  id          String  @id @default(cuid())
  projectId   String
  funderAddress String
  amount      Decimal @db.Decimal(18, 4)
  txHash      String  @unique
  fundedAt    DateTime @default(now())

  // Relations
  project     Project @relation(fields: [projectId], references: [id])

  @@map("project_fundings")
}

model KnowledgeToken {
  id            String      @id @default(cuid())
  tokenId       Int         @unique
  contractAddress String
  title         String
  description   String
  tokenType     TokenType
  ipfsHash      String
  creatorId     String
  consultantId  String?
  daoId         String?
  price         Decimal     @db.Decimal(18, 4)
  accessCount   Int         @default(0)
  isForSale     Boolean     @default(true)
  createdAt     DateTime    @default(now())

  // Relations
  creator       User        @relation(fields: [creatorId], references: [id])
  consultant    ConsultantProfile? @relation(fields: [consultantId], references: [id])
  dao           AgriDAO?    @relation(fields: [daoId], references: [id])
  purchases     TokenPurchase[]

  @@map("knowledge_tokens")
}

model TokenPurchase {
  id          String   @id @default(cuid())
  tokenId     String
  buyerAddress String
  price       Decimal  @db.Decimal(18, 4)
  txHash      String   @unique
  purchasedAt DateTime @default(now())

  // Relations
  token       KnowledgeToken @relation(fields: [tokenId], references: [id])

  @@map("token_purchases")
}

model Consultation {
  id              String            @id @default(cuid())
  consultantId    String
  clientAddress   String
  daoId           String?
  topic           String
  description     String?
  scheduledAt     DateTime
  duration        Int               @default(60) // minutes
  fee             Decimal           @db.Decimal(18, 4)
  status          ConsultationStatus @default(SCHEDULED)
  meetingUrl      String?
  recordingUrl    String?
  summary         String?
  createdAt       DateTime          @default(now())

  // Relations
  consultant      ConsultantProfile @relation(fields: [consultantId], references: [id])
  dao             AgriDAO?          @relation(fields: [daoId], references: [id])

  @@map("consultations")
}

model Supplier {
  id              String        @id @default(cuid())
  walletAddress   String        @unique
  name            String
  businessLicense String
  certifications  String[]
  status          SupplierStatus @default(UNVERIFIED)
  verifiedAt      DateTime?
  verifierId      String?
  createdAt       DateTime      @default(now())

  // Relations
  reviews         Review[]

  @@map("suppliers")
}

model Review {
  id            String   @id @default(cuid())
  supplierId    String
  reviewerId    String
  rating        Int      // 1-5
  comment       String?
  isVerified    Boolean  @default(false)
  txHash        String?  @unique
  createdAt     DateTime @default(now())

  // Relations
  supplier      Supplier @relation(fields: [supplierId], references: [id])
  reviewer      User     @relation(fields: [reviewerId], references: [id])

  @@map("reviews")
}

// Enums
enum TokenType {
  GUIDE
  CONSULTATION
  DATA
  EXPERIENCE
}

enum ConsultationStatus {
  SCHEDULED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

enum SupplierStatus {
  UNVERIFIED
  VERIFIED
  BLACKLISTED
}
```

## 🔗 API Architecture

### Express.js REST API

```typescript
// src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { PrismaClient } from '@prisma/client';
import { createServer } from 'http';
import { Server } from 'socket.io';

// Routes
import authRoutes from './api/auth';
import agriDAORoutes from './api/agridaos';
import consultantRoutes from './api/consultants';
import knowledgeRoutes from './api/knowledge';
import blockchainRoutes from './api/blockchain';

// Middleware
import { authMiddleware } from './middleware/auth';
import { validationMiddleware } from './middleware/validation';

const app = express();
const server = createServer(app);
const io = new Server(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    methods: ["GET", "POST"]
  }
});

// Database
export const prisma = new PrismaClient();

// Middleware
app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true
}));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use(limiter);

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/agridaos', authMiddleware, agriDAORoutes);
app.use('/api/consultants', consultantRoutes);
app.use('/api/knowledge', authMiddleware, knowledgeRoutes);
app.use('/api/blockchain', blockchainRoutes);

// WebSocket setup
io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  socket.on('join-dao', (daoId) => {
    socket.join(`dao-${daoId}`);
  });

  socket.on('leave-dao', (daoId) => {
    socket.leave(`dao-${daoId}`);
  });

  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});

export { io };

const PORT = process.env.PORT || 3001;
server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### AgriDAO API Routes

```typescript
// src/api/agridaos/index.ts
import { Router } from 'express';
import { prisma } from '../../app';
import { BlockchainService } from '../../services/BlockchainService';
import { z } from 'zod';
import { validationMiddleware } from '../../middleware/validation';

const router = Router();
const blockchainService = new BlockchainService();

// Get all AgriDAOs
router.get('/', async (req, res) => {
  try {
    const daos = await prisma.agriDAO.findMany({
      include: {
        members: {
          include: {
            user: {
              select: { id: true, name: true, walletAddress: true }
            }
          }
        },
        projects: {
          where: { isActive: true }
        },
        _count: {
          select: { members: true, projects: true }
        }
      },
      orderBy: { createdAt: 'desc' }
    });

    res.json({
      success: true,
      data: daos
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch AgriDAOs'
    });
  }
});

// Get specific AgriDAO
router.get('/:id', async (req, res) => {
  try {
    const dao = await prisma.agriDAO.findUnique({
      where: { id: req.params.id },
      include: {
        members: {
          include: {
            user: {
              select: { id: true, name: true, walletAddress: true, avatar: true }
            }
          }
        },
        projects: {
          include: {
            fundings: true
          }
        },
        consultations: {
          include: {
            consultant: {
              include: {
                user: {
                  select: { name: true, avatar: true }
                }
              }
            }
          }
        }
      }
    });

    if (!dao) {
      return res.status(404).json({
        success: false,
        error: 'AgriDAO not found'
      });
    }

    res.json({
      success: true,
      data: dao
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to fetch AgriDAO'
    });
  }
});

// Create project proposal
const createProjectSchema = z.object({
  daoId: z.string(),
  title: z.string().min(5),
  description: z.string().min(20),
  fundingGoal: z.string(),
  deadline: z.string().datetime(),
});

router.post('/projects', validationMiddleware(createProjectSchema), async (req, res) => {
  try {
    const { daoId, title, description, fundingGoal, deadline } = req.body;
    const userAddress = req.user.walletAddress;

    // Verify user is member of DAO
    const membership = await prisma.agriDAOMember.findFirst({
      where: {
        daoId,
        user: { walletAddress: userAddress }
      }
    });

    if (!membership) {
      return res.status(403).json({
        success: false,
        error: 'Not a member of this DAO'
      });
    }

    const project = await prisma.project.create({
      data: {
        daoId,
        title,
        description,
        fundingGoal: parseFloat(fundingGoal),
        proposerId: userAddress,
        deadline: new Date(deadline)
      }
    });

    res.json({
      success: true,
      data: project
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Failed to create project'
    });
  }
});

// Sync blockchain events
router.post('/sync/:contractAddress', async (req, res) => {
  try {
    const { contractAddress } = req.params;
    
    await blockchainService.syncAgriDAOEvents(contractAddress);
    
    res.json({
      success: true,
      message: 'Sync completed'
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Sync failed'
    });
  }
});

export default router;
```

## 🤖 AI Services Architecture

### Python FastAPI AI Service

```python
# ai-services/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import asyncio
import logging

from agents.consultant_agent import ConsultantAgent
from agents.diagnosis_agent import DiagnosisAgent
from agents.market_agent import MarketAgent
from models.nlp_model import NLPModel
from models.image_classifier import ImageClassifier
from models.price_predictor import PricePredictor

app = FastAPI(title="Optyma AI Services", version="1.0.0")

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure properly for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize AI models
nlp_model = NLPModel()
image_classifier = ImageClassifier()
price_predictor = PricePredictor()

# Initialize agents
consultant_agent = ConsultantAgent(nlp_model)
diagnosis_agent = DiagnosisAgent(image_classifier, nlp_model)
market_agent = MarketAgent(price_predictor)

# Pydantic models for requests
class ConsultationRequest(BaseModel):
    question: str
    specialty: str  # "aquaculture", "livestock", "crops"
    context: Optional[str] = None
    user_id: str

class DiagnosisRequest(BaseModel):
    image_url: Optional[str] = None
    symptoms: str
    crop_type: str
    user_id: str

class MarketAnalysisRequest(BaseModel):
    commodity: str
    region: str
    time_horizon: int  # days
    user_id: str

class ConsultationResponse(BaseModel):
    answer: str
    confidence: float
    sources: List[str]
    follow_up_questions: List[str]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": "2025-07-28T00:00:00Z"}

@app.post("/consultation", response_model=ConsultationResponse)
async def get_consultation(request: ConsultationRequest):
    """
    Get AI-powered agricultural consultation
    """
    try:
        response = await consultant_agent.process_question(
            question=request.question,
            specialty=request.specialty,
            context=request.context
        )
        
        return ConsultationResponse(
            answer=response["answer"],
            confidence=response["confidence"],
            sources=response["sources"],
            follow_up_questions=response["follow_up_questions"]
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/diagnosis")
async def get_diagnosis(request: DiagnosisRequest):
    """
    Diagnose crop/livestock diseases
    """
    try:
        diagnosis = await diagnosis_agent.diagnose(
            image_url=request.image_url,
            symptoms=request.symptoms,
            crop_type=request.crop_type
        )
        
        return {
            "diagnosis": diagnosis["disease"],
            "confidence": diagnosis["confidence"],
            "treatment": diagnosis["treatment"],
            "prevention": diagnosis["prevention"],
            "severity": diagnosis["severity"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/market-analysis")
async def get_market_analysis(request: MarketAnalysisRequest):
    """
    Get market price predictions and analysis
    """
    try:
        analysis = await market_agent.analyze_market(
            commodity=request.commodity,
            region=request.region,
            time_horizon=request.time_horizon
        )
        
        return {
            "current_price": analysis["current_price"],
            "predicted_price": analysis["predicted_price"],
            "price_trend": analysis["trend"],
            "confidence": analysis["confidence"],
            "factors": analysis["influencing_factors"],
            "recommendations": analysis["recommendations"]
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Consultant Agent Implementation

```python
# ai-services/agents/consultant_agent.py
import asyncio
import json
from typing import Dict, List, Optional
from langchain.llms import OpenAI
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Pinecone
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory

class ConsultantAgent:
    def __init__(self, nlp_model):
        self.nlp_model = nlp_model
        self.llm = OpenAI(temperature=0.7)
        self.embeddings = OpenAIEmbeddings()
        
        # Initialize knowledge base (25 years of Optyma experience)
        self.knowledge_base = self._initialize_knowledge_base()
        
        # Specialty-specific chains
        self.chains = {
            "aquaculture": self._create_specialty_chain("aquaculture"),
            "livestock": self._create_specialty_chain("livestock"),
            "crops": self._create_specialty_chain("crops"),
            "probiotics": self._create_specialty_chain("probiotics")
        }

    def _initialize_knowledge_base(self):
        """Initialize the knowledge base with Optyma's 25 years of experience"""
        # This would connect to Pinecone vector database
        # containing all agricultural knowledge, best practices, etc.
        pass

    def _create_specialty_chain(self, specialty: str):
        """Create a conversational chain for specific agricultural specialty"""
        memory = ConversationBufferMemory(
            memory_key="chat_history",
            return_messages=True
        )
        
        return ConversationalRetrievalChain.from_llm(
            llm=self.llm,
            retriever=self.knowledge_base.as_retriever(
                search_kwargs={"filter": {"specialty": specialty}}
            ),
            memory=memory,
            verbose=True
        )

    async def process_question(
        self, 
        question: str, 
        specialty: str, 
        context: Optional[str] = None
    ) -> Dict:
        """Process agricultural consultation question"""
        
        # Get the appropriate chain for the specialty
        chain = self.chains.get(specialty)
        if not chain:
            raise ValueError(f"Unsupported specialty: {specialty}")
        
        # Enhance question with context if provided
        enhanced_question = question
        if context:
            enhanced_question = f"Context: {context}\n\nQuestion: {question}"
        
        # Get response from the chain
        response = await asyncio.to_thread(
            chain.run, {"question": enhanced_question}
        )
        
        # Extract sources and generate follow-up questions
        sources = self._extract_sources(response)
        follow_ups = self._generate_follow_up_questions(question, specialty)
        confidence = self._calculate_confidence(response, question)
        
        return {
            "answer": response,
            "confidence": confidence,
            "sources": sources,
            "follow_up_questions": follow_ups
        }

    def _extract_sources(self, response: str) -> List[str]:
        """Extract source references from the response"""
        # Implementation to extract sources from the knowledge base
        return ["25 Years Optyma Experience", "Best Practices Database"]

    def _generate_follow_up_questions(self, question: str, specialty: str) -> List[str]:
        """Generate relevant follow-up questions"""
        follow_ups = {
            "aquaculture": [
                "Apakah Anda ingin informasi tentang kualitas air?",
                "Bagaimana kondisi pakan udang saat ini?",
                "Apakah ada gejala penyakit yang terlihat?"
            ],
            "livestock": [
                "Bagaimana kondisi kandang saat ini?",
                "Apakah sudah menggunakan probiotik?",
                "Bagaimana nafsu makan ternak?"
            ],
            "crops": [
                "Bagaimana kondisi tanah di lahan Anda?",
                "Apakah menggunakan pupuk organik?",
                "Bagaimana sistem irigasi yang digunakan?"
            ]
        }
        
        return follow_ups.get(specialty, ["Ada pertanyaan lain?"])

    def _calculate_confidence(self, response: str, question: str) -> float:
        """Calculate confidence score for the response"""
        # Simple confidence calculation based on response length and keyword matching
        base_confidence = 0.7
        
        # Adjust based on response quality indicators
        if len(response) > 100:
            base_confidence += 0.1
        if "berdasarkan pengalaman" in response.lower():
            base_confidence += 0.1
        if "rekomendasi" in response.lower():
            base_confidence += 0.05
            
        return min(base_confidence, 0.95)
```

### Blockchain Integration Service

```typescript
// src/services/BlockchainService.ts
import { ethers } from 'ethers';
import { prisma } from '../app';
import { io } from '../app';

export class BlockchainService {
  private provider: ethers.Provider;
  private contracts: Map<string, ethers.Contract>;

  constructor() {
    this.provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    this.contracts = new Map();
    this.initializeContracts();
  }

  private async initializeContracts() {
    // Initialize contract instances
    const agriDAOFactoryABI = await import('../contracts/AgriDAOFactory.json');
    const knowledgeTokensABI = await import('../contracts/KnowledgeTokens.json');

    // Add contracts to map
    this.contracts.set('AgriDAOFactory', new ethers.Contract(
      process.env.AGRIDAO_FACTORY_ADDRESS!,
      agriDAOFactoryABI.abi,
      this.provider
    ));

    this.contracts.set('KnowledgeTokens', new ethers.Contract(
      process.env.KNOWLEDGE_TOKENS_ADDRESS!,
      knowledgeTokensABI.abi,
      this.provider
    ));

    // Setup event listeners
    this.setupEventListeners();
  }

  private setupEventListeners() {
    const factory = this.contracts.get('AgriDAOFactory');
    const knowledgeTokens = this.contracts.get('KnowledgeTokens');

    // Listen for new DAO creation
    factory?.on('DAOCreated', async (daoId, daoAddress, specialty, event) => {
      await this.handleDAOCreated(daoId, daoAddress, specialty, event);
    });

    // Listen for knowledge token minting
    knowledgeTokens?.on('KnowledgeTokenMinted', async (tokenId, creator, tokenType, event) => {
      await this.handleKnowledgeTokenMinted(tokenId, creator, tokenType, event);
    });
  }

  private async handleDAOCreated(
    daoId: bigint, 
    daoAddress: string, 
    specialty: string, 
    event: ethers.Log
  ) {
    try {
      // Get transaction details
      const tx = await this.provider.getTransaction(event.transactionHash);
      
      // Create DAO record in database
      const dao = await prisma.agriDAO.create({
        data: {
          contractAddress: daoAddress,
          // Additional data would be fetched from contract
          name: 'New AgriDAO', // This would come from contract
          symbol: 'AGRI', // This would come from contract
          specialty,
          targetRaise: 0, // This would come from contract
        }
      });

      // Notify connected clients
      io.to('dao-updates').emit('dao-created', {
        dao,
        transactionHash: event.transactionHash
      });
    } catch (error) {
      console.error('Error handling DAO creation:', error);
    }
  }

  private async handleKnowledgeTokenMinted(
    tokenId: bigint,
    creator: string,
    tokenType: string,
    event: ethers.Log
  ) {
    try {
      // Get token metadata from contract
      const knowledgeContract = this.contracts.get('KnowledgeTokens');
      const tokenData = await knowledgeContract?.knowledgeTokens(tokenId);
      
      // Find or create user
      let user = await prisma.user.findUnique({
        where: { walletAddress: creator }
      });

      if (!user) {
        user = await prisma.user.create({
          data: { walletAddress: creator }
        });
      }

      // Create knowledge token record
      const knowledgeToken = await prisma.knowledgeToken.create({
        data: {
          tokenId: Number(tokenId),
          contractAddress: process.env.KNOWLEDGE_TOKENS_ADDRESS!,
          title: tokenData.title,
          description: tokenData.description,
          tokenType: tokenType.toUpperCase() as any,
          ipfsHash: tokenData.ipfsHash,
          creatorId: user.id,
          price: parseFloat(ethers.formatEther(tokenData.price))
        }
      });

      // Notify connected clients
      io.emit('knowledge-token-minted', {
        token: knowledgeToken,
        transactionHash: event.transactionHash
      });
    } catch (error) {
      console.error('Error handling knowledge token minting:', error);
    }
  }

  async syncAgriDAOEvents(contractAddress: string) {
    // Implementation to sync all historical events for a DAO
    const daoContract = new ethers.Contract(
      contractAddress,
      (await import('../contracts/AgriDAO.json')).abi,
      this.provider
    );

    // Get all historical events and sync to database
    const filter = daoContract.filters.ProjectProposed();
    const events = await daoContract.queryFilter(filter);

    for (const event of events) {
      // Process each event and update database
      await this.handleProjectProposed(event);
    }
  }

  private async handleProjectProposed(event: ethers.Log) {
    // Implementation to handle project proposal events
    // Update database, notify clients, etc.
  }
}
```

## 📊 Performance Monitoring

### Logging & Monitoring Setup

```typescript
// src/utils/logger.ts
import winston from 'winston';

export const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'optyma-backend' },
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// Performance monitoring middleware
export const performanceMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('Request completed', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.get('User-Agent')
    });
  });
  
  next();
};
```

---

*Dokumen ini mencakup backend dan AI architecture. Untuk deployment dan scaling strategies, lihat dokumen terpisah.*
