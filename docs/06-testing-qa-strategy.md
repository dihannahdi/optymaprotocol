# OPTYMA AGRITECH PROTOCOL - Testing & QA Strategy

## 🧪 Strategi Testing Komprehensif

### Overview Testing Strategy

```
Testing Pyramid untuk Optyma Agritech Protocol

                    ┌─────────────────┐
                    │   E2E Tests     │ ← 10% (Critical user journeys)
                    │   (Playwright)  │
                ┌───┴─────────────────┴───┐
                │  Integration Tests      │ ← 20% (API + Blockchain)
                │  (Jest + Supertest)     │
        ┌───────┴─────────────────────────┴───────┐
        │          Unit Tests                     │ ← 70% (Functions + Components)
        │    (Jest + React Testing Library)      │
        └─────────────────────────────────────────┘
```

## 🔧 Unit Testing Strategy

### Smart Contract Testing

```javascript
// tests/smart-contracts/OptimaToken.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("OptimaToken Contract", function () {
  let optimaToken;
  let owner;
  let user1;
  let user2;

  beforeEach(async function () {
    [owner, user1, user2] = await ethers.getSigners();
    
    const OptimaToken = await ethers.getContractFactory("OptimaToken");
    optimaToken = await OptimaToken.deploy();
    await optimaToken.deployed();
  });

  describe("Token Distribution", function () {
    it("Should mint initial supply to owner", async function () {
      const ownerBalance = await optimaToken.balanceOf(owner.address);
      expect(ownerBalance).to.equal(ethers.utils.parseEther("1000000"));
    });

    it("Should transfer tokens correctly", async function () {
      const transferAmount = ethers.utils.parseEther("1000");
      
      await optimaToken.transfer(user1.address, transferAmount);
      
      const user1Balance = await optimaToken.balanceOf(user1.address);
      expect(user1Balance).to.equal(transferAmount);
    });

    it("Should fail when transferring more than balance", async function () {
      const transferAmount = ethers.utils.parseEther("2000000");
      
      await expect(
        optimaToken.transfer(user1.address, transferAmount)
      ).to.be.revertedWith("ERC20: transfer amount exceeds balance");
    });
  });

  describe("Staking Functionality", function () {
    it("Should allow staking of tokens", async function () {
      const stakeAmount = ethers.utils.parseEther("100");
      
      await optimaToken.stake(stakeAmount);
      
      const stakedBalance = await optimaToken.stakedBalance(owner.address);
      expect(stakedBalance).to.equal(stakeAmount);
    });

    it("Should calculate rewards correctly", async function () {
      const stakeAmount = ethers.utils.parseEther("100");
      
      await optimaToken.stake(stakeAmount);
      
      // Simulate time passage (30 days)
      await ethers.provider.send("evm_increaseTime", [30 * 24 * 60 * 60]);
      await ethers.provider.send("evm_mine");
      
      const rewards = await optimaToken.calculateRewards(owner.address);
      expect(rewards).to.be.gt(0);
    });
  });
});

// tests/smart-contracts/AgriDAO.test.js
describe("AgriDAO Factory", function () {
  let agriDAOFactory;
  let optimaToken;
  let owner;
  let farmer1;

  beforeEach(async function () {
    [owner, farmer1] = await ethers.getSigners();
    
    // Deploy dependencies
    const OptimaToken = await ethers.getContractFactory("OptimaToken");
    optimaToken = await OptimaToken.deploy();
    
    const AgriDAOFactory = await ethers.getContractFactory("AgriDAOFactory");
    agriDAOFactory = await AgriDAOFactory.deploy(optimaToken.address);
  });

  it("Should create new AgriDAO correctly", async function () {
    const daoName = "Rice Farmers Cooperative";
    const minStake = ethers.utils.parseEther("100");
    
    const tx = await agriDAOFactory.createAgriDAO(
      daoName,
      minStake,
      farmer1.address
    );
    
    const receipt = await tx.wait();
    const event = receipt.events.find(e => e.event === 'AgriDAOCreated');
    
    expect(event.args.name).to.equal(daoName);
    expect(event.args.creator).to.equal(farmer1.address);
  });

  it("Should require minimum stake for DAO creation", async function () {
    const daoName = "Invalid DAO";
    const minStake = ethers.utils.parseEther("0");
    
    await expect(
      agriDAOFactory.createAgriDAO(daoName, minStake, farmer1.address)
    ).to.be.revertedWith("Minimum stake must be greater than 0");
  });
});
```

### Backend API Testing

```javascript
// tests/api/agridao.test.js
const request = require('supertest');
const app = require('../../src/app');
const { PrismaClient } = require('@prisma/client');

const prisma = new PrismaClient();

describe('AgriDAO API Endpoints', () => {
  beforeEach(async () => {
    // Clean database before each test
    await prisma.agriDAO.deleteMany();
    await prisma.user.deleteMany();
  });

  afterAll(async () => {
    await prisma.$disconnect();
  });

  describe('POST /api/agridao', () => {
    it('should create new AgriDAO', async () => {
      const agriDAOData = {
        name: 'Test Rice DAO',
        description: 'Testing DAO creation',
        category: 'rice',
        minimumStake: '100',
        walletAddress: '0x1234567890123456789012345678901234567890'
      };

      const response = await request(app)
        .post('/api/agridao')
        .send(agriDAOData)
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.data.name).toBe(agriDAOData.name);
      expect(response.body.data.category).toBe(agriDAOData.category);
    });

    it('should validate required fields', async () => {
      const invalidData = {
        description: 'Missing required fields'
      };

      const response = await request(app)
        .post('/api/agridao')
        .send(invalidData)
        .expect(400);

      expect(response.body.success).toBe(false);
      expect(response.body.errors).toContain('Name is required');
    });

    it('should validate wallet address format', async () => {
      const invalidData = {
        name: 'Test DAO',
        description: 'Test description',
        category: 'rice',
        minimumStake: '100',
        walletAddress: 'invalid-address'
      };

      const response = await request(app)
        .post('/api/agridao')
        .send(invalidData)
        .expect(400);

      expect(response.body.errors).toContain('Invalid wallet address format');
    });
  });

  describe('GET /api/agridao', () => {
    beforeEach(async () => {
      // Create test data
      await prisma.agriDAO.createMany({
        data: [
          {
            name: 'Rice DAO 1',
            description: 'First rice DAO',
            category: 'rice',
            minimumStake: '100',
            walletAddress: '0x1111111111111111111111111111111111111111',
            isActive: true
          },
          {
            name: 'Corn DAO 1',
            description: 'First corn DAO',
            category: 'corn',
            minimumStake: '200',
            walletAddress: '0x2222222222222222222222222222222222222222',
            isActive: true
          }
        ]
      });
    });

    it('should return list of AgriDAOs', async () => {
      const response = await request(app)
        .get('/api/agridao')
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data).toHaveLength(2);
      expect(response.body.data[0].name).toBe('Rice DAO 1');
    });

    it('should filter by category', async () => {
      const response = await request(app)
        .get('/api/agridao?category=rice')
        .expect(200);

      expect(response.body.data).toHaveLength(1);
      expect(response.body.data[0].category).toBe('rice');
    });

    it('should paginate results', async () => {
      const response = await request(app)
        .get('/api/agridao?page=1&limit=1')
        .expect(200);

      expect(response.body.data).toHaveLength(1);
      expect(response.body.pagination.page).toBe(1);
      expect(response.body.pagination.total).toBe(2);
    });
  });
});

// tests/api/auth.test.js
describe('Authentication Middleware', () => {
  it('should authenticate valid JWT token', async () => {
    const token = jwt.sign(
      { walletAddress: '0x1234567890123456789012345678901234567890' },
      process.env.JWT_SECRET
    );

    const response = await request(app)
      .get('/api/profile')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(response.body.success).toBe(true);
  });

  it('should reject invalid JWT token', async () => {
    const response = await request(app)
      .get('/api/profile')
      .set('Authorization', 'Bearer invalid-token')
      .expect(401);

    expect(response.body.error).toBe('Invalid token');
  });

  it('should require authentication for protected routes', async () => {
    const response = await request(app)
      .get('/api/profile')
      .expect(401);

    expect(response.body.error).toBe('No token provided');
  });
});
```

### Frontend Component Testing

```javascript
// tests/components/AgriDAOCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { AgriDAOCard } from '../../src/components/AgriDAOCard';

const mockAgriDAO = {
  id: '1',
  name: 'Rice Farmers Cooperative',
  description: 'A cooperative for rice farmers in Central Java',
  category: 'rice',
  memberCount: 25,
  minimumStake: '100',
  totalStaked: '5000',
  walletAddress: '0x1234567890123456789012345678901234567890',
  isActive: true,
  createdAt: new Date('2025-01-01')
};

describe('AgriDAOCard Component', () => {
  it('renders AgriDAO information correctly', () => {
    render(<AgriDAOCard agriDAO={mockAgriDAO} />);

    expect(screen.getByText('Rice Farmers Cooperative')).toBeInTheDocument();
    expect(screen.getByText('A cooperative for rice farmers in Central Java')).toBeInTheDocument();
    expect(screen.getByText('25 Members')).toBeInTheDocument();
    expect(screen.getByText('100 OPTYMA')).toBeInTheDocument();
  });

  it('shows join button for non-members', () => {
    render(<AgriDAOCard agriDAO={mockAgriDAO} isMember={false} />);

    const joinButton = screen.getByText('Join DAO');
    expect(joinButton).toBeInTheDocument();
    expect(joinButton).not.toBeDisabled();
  });

  it('shows member badge for members', () => {
    render(<AgriDAOCard agriDAO={mockAgriDAO} isMember={true} />);

    expect(screen.getByText('Member')).toBeInTheDocument();
    expect(screen.queryByText('Join DAO')).not.toBeInTheDocument();
  });

  it('calls onJoin when join button is clicked', () => {
    const mockOnJoin = jest.fn();
    render(
      <AgriDAOCard 
        agriDAO={mockAgriDAO} 
        isMember={false} 
        onJoin={mockOnJoin} 
      />
    );

    const joinButton = screen.getByText('Join DAO');
    fireEvent.click(joinButton);

    expect(mockOnJoin).toHaveBeenCalledWith(mockAgriDAO.id);
  });

  it('displays correct category badge', () => {
    render(<AgriDAOCard agriDAO={mockAgriDAO} />);

    const categoryBadge = screen.getByText('Rice');
    expect(categoryBadge).toBeInTheDocument();
    expect(categoryBadge).toHaveClass('bg-green-100', 'text-green-800');
  });
});

// tests/hooks/useAgriDAO.test.tsx
import { renderHook, waitFor } from '@testing-library/react';
import { useAgriDAO } from '../../src/hooks/useAgriDAO';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  });
  
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

// Mock API
const mockApi = {
  getAgriDAOs: jest.fn(),
  createAgriDAO: jest.fn(),
  joinAgriDAO: jest.fn()
};

jest.mock('../../src/lib/api', () => mockApi);

describe('useAgriDAO Hook', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('fetches AgriDAOs successfully', async () => {
    const mockData = [mockAgriDAO];
    mockApi.getAgriDAOs.mockResolvedValue({ data: mockData });

    const { result } = renderHook(() => useAgriDAO(), {
      wrapper: createWrapper()
    });

    await waitFor(() => {
      expect(result.current.agriDAOs).toEqual(mockData);
      expect(result.current.isLoading).toBe(false);
    });
  });

  it('handles API errors correctly', async () => {
    const errorMessage = 'Failed to fetch AgriDAOs';
    mockApi.getAgriDAOs.mockRejectedValue(new Error(errorMessage));

    const { result } = renderHook(() => useAgriDAO(), {
      wrapper: createWrapper()
    });

    await waitFor(() => {
      expect(result.current.error).toBeTruthy();
      expect(result.current.isLoading).toBe(false);
    });
  });

  it('creates new AgriDAO successfully', async () => {
    const newAgriDAO = { ...mockAgriDAO, id: '2', name: 'New DAO' };
    mockApi.createAgriDAO.mockResolvedValue({ data: newAgriDAO });

    const { result } = renderHook(() => useAgriDAO(), {
      wrapper: createWrapper()
    });

    await waitFor(() => {
      result.current.createAgriDAO.mutate({
        name: 'New DAO',
        description: 'Test description',
        category: 'rice',
        minimumStake: '100'
      });
    });

    await waitFor(() => {
      expect(result.current.createAgriDAO.isSuccess).toBe(true);
    });
  });
});
```

## 🔄 Integration Testing

### API Integration Tests

```javascript
// tests/integration/blockchain-api.test.js
const { ethers } = require('hardhat');
const request = require('supertest');
const app = require('../../src/app');

describe('Blockchain API Integration', () => {
  let optimaToken;
  let agriDAOFactory;
  let owner;
  let user1;

  beforeAll(async () => {
    // Deploy contracts
    [owner, user1] = await ethers.getSigners();
    
    const OptimaToken = await ethers.getContractFactory("OptimaToken");
    optimaToken = await OptimaToken.deploy();
    
    const AgriDAOFactory = await ethers.getContractFactory("AgriDAOFactory");
    agriDAOFactory = await AgriDAOFactory.deploy(optimaToken.address);

    // Update API with contract addresses
    process.env.OPTIMA_TOKEN_ADDRESS = optimaToken.address;
    process.env.AGRIDAO_FACTORY_ADDRESS = agriDAOFactory.address;
  });

  it('should sync blockchain events with database', async () => {
    // Create AgriDAO on blockchain
    const tx = await agriDAOFactory.createAgriDAO(
      "Integration Test DAO",
      ethers.utils.parseEther("100"),
      user1.address
    );
    
    const receipt = await tx.wait();
    const event = receipt.events.find(e => e.event === 'AgriDAOCreated');

    // Wait for event processing
    await new Promise(resolve => setTimeout(resolve, 2000));

    // Check if DAO was created in database
    const response = await request(app)
      .get('/api/agridao')
      .expect(200);

    const dao = response.body.data.find(d => d.name === "Integration Test DAO");
    expect(dao).toBeTruthy();
    expect(dao.walletAddress.toLowerCase()).toBe(event.args.daoAddress.toLowerCase());
  });

  it('should handle token staking through API', async () => {
    // Transfer tokens to user1
    await optimaToken.transfer(user1.address, ethers.utils.parseEther("1000"));

    // Stake tokens through API
    const stakeData = {
      amount: "100",
      walletAddress: user1.address
    };

    const response = await request(app)
      .post('/api/stake')
      .send(stakeData)
      .expect(200);

    expect(response.body.success).toBe(true);

    // Verify staking on blockchain
    const stakedBalance = await optimaToken.stakedBalance(user1.address);
    expect(stakedBalance).to.equal(ethers.utils.parseEther("100"));
  });
});

// tests/integration/ai-api.test.js
describe('AI Service Integration', () => {
  it('should process consultation request end-to-end', async () => {
    const consultationData = {
      type: 'crop_diagnosis',
      cropType: 'rice',
      symptoms: ['yellowing leaves', 'stunted growth'],
      images: ['base64-encoded-image-data'],
      location: 'Central Java, Indonesia'
    };

    const response = await request(app)
      .post('/api/consultation')
      .send(consultationData)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(response.body.data.diagnosis).toBeTruthy();
    expect(response.body.data.confidence).toBeGreaterThan(0.7);
    expect(response.body.data.recommendations).toHaveLength.greaterThan(0);
  });

  it('should handle image classification correctly', async () => {
    const imageData = {
      image: 'base64-encoded-rice-disease-image',
      cropType: 'rice'
    };

    const response = await request(app)
      .post('/api/ai/classify-disease')
      .send(imageData)
      .expect(200);

    expect(response.body.disease).toBeTruthy();
    expect(response.body.confidence).toBeGreaterThan(0.8);
    expect(response.body.treatment).toBeTruthy();
  });
});
```

### WebSocket Integration Tests

```javascript
// tests/integration/websocket.test.js
const io = require('socket.io-client');
const { createServer } = require('../../src/server');

describe('WebSocket Integration', () => {
  let server;
  let clientSocket;

  beforeAll((done) => {
    server = createServer();
    server.listen(() => {
      const port = server.address().port;
      clientSocket = io(`http://localhost:${port}`);
      clientSocket.on('connect', done);
    });
  });

  afterAll(() => {
    server.close();
    clientSocket.close();
  });

  it('should broadcast AgriDAO updates', (done) => {
    clientSocket.on('agridao:updated', (data) => {
      expect(data.id).toBeTruthy();
      expect(data.name).toBeTruthy();
      done();
    });

    // Trigger AgriDAO update
    clientSocket.emit('agridao:subscribe', { id: 'test-dao-id' });
    
    // Simulate update from another source
    setTimeout(() => {
      clientSocket.emit('agridao:update', {
        id: 'test-dao-id',
        name: 'Updated DAO Name'
      });
    }, 100);
  });

  it('should handle consultation status updates', (done) => {
    const consultationId = 'test-consultation-123';

    clientSocket.on('consultation:status', (data) => {
      expect(data.id).toBe(consultationId);
      expect(data.status).toBe('completed');
      done();
    });

    clientSocket.emit('consultation:subscribe', { id: consultationId });

    // Simulate consultation completion
    setTimeout(() => {
      clientSocket.emit('consultation:complete', {
        id: consultationId,
        status: 'completed'
      });
    }, 100);
  });
});
```

## 🎭 End-to-End Testing

### Playwright E2E Tests

```javascript
// tests/e2e/agridao-journey.spec.js
const { test, expect } = require('@playwright/test');

test.describe('AgriDAO User Journey', () => {
  test.beforeEach(async ({ page }) => {
    // Mock MetaMask connection
    await page.addInitScript(() => {
      window.ethereum = {
        isMetaMask: true,
        request: async ({ method }) => {
          if (method === 'eth_requestAccounts') {
            return ['0x1234567890123456789012345678901234567890'];
          }
          if (method === 'eth_chainId') {
            return '0x89'; // Polygon mainnet
          }
          return null;
        },
        on: () => {},
        removeListener: () => {}
      };
    });

    await page.goto('/');
  });

  test('Complete AgriDAO creation flow', async ({ page }) => {
    // Connect wallet
    await page.click('[data-testid="connect-wallet"]');
    await page.click('[data-testid="metamask-option"]');
    
    // Wait for connection
    await expect(page.locator('[data-testid="wallet-address"]')).toBeVisible();

    // Navigate to create DAO
    await page.click('[data-testid="create-dao-button"]');
    
    // Fill DAO creation form
    await page.fill('[data-testid="dao-name"]', 'E2E Test Rice DAO');
    await page.fill('[data-testid="dao-description"]', 'This is a test DAO created by E2E tests');
    await page.selectOption('[data-testid="dao-category"]', 'rice');
    await page.fill('[data-testid="minimum-stake"]', '100');

    // Submit form
    await page.click('[data-testid="create-dao-submit"]');

    // Wait for transaction confirmation
    await expect(page.locator('[data-testid="transaction-pending"]')).toBeVisible();
    await expect(page.locator('[data-testid="transaction-success"]')).toBeVisible({ timeout: 30000 });

    // Verify DAO appears in list
    await page.goto('/agridaos');
    await expect(page.locator('text=E2E Test Rice DAO')).toBeVisible();
  });

  test('Join existing AgriDAO', async ({ page }) => {
    // Navigate to AgriDAO list
    await page.goto('/agridaos');

    // Find and join a DAO
    const daoCard = page.locator('[data-testid="agridao-card"]').first();
    await daoCard.locator('[data-testid="join-dao-button"]').click();

    // Confirm stake amount
    await page.fill('[data-testid="stake-amount"]', '150');
    await page.click('[data-testid="confirm-join"]');

    // Wait for transaction
    await expect(page.locator('[data-testid="transaction-success"]')).toBeVisible({ timeout: 30000 });

    // Verify membership
    await page.reload();
    await expect(daoCard.locator('[data-testid="member-badge"]')).toBeVisible();
  });

  test('Request AI consultation', async ({ page }) => {
    // Navigate to consultation page
    await page.goto('/consultation');

    // Fill consultation form
    await page.selectOption('[data-testid="crop-type"]', 'rice');
    await page.fill('[data-testid="symptoms"]', 'Yellow leaves and poor growth');
    await page.fill('[data-testid="location"]', 'Central Java');

    // Upload image (mock file)
    const fileInput = page.locator('[data-testid="image-upload"]');
    await fileInput.setInputFiles({
      name: 'rice-disease.jpg',
      mimeType: 'image/jpeg',
      buffer: Buffer.from('fake-image-data')
    });

    // Submit consultation
    await page.click('[data-testid="submit-consultation"]');

    // Wait for AI response
    await expect(page.locator('[data-testid="ai-response"]')).toBeVisible({ timeout: 10000 });
    await expect(page.locator('[data-testid="diagnosis"]')).toBeVisible();
    await expect(page.locator('[data-testid="recommendations"]')).toBeVisible();
  });
});

// tests/e2e/responsive.spec.js
test.describe('Responsive Design', () => {
  const devices = ['iPhone 12', 'iPad', 'Desktop'];

  devices.forEach(device => {
    test(`Should work on ${device}`, async ({ page, browser }) => {
      const context = await browser.newContext({
        ...devices[device]
      });
      const responsivePage = await context.newPage();

      await responsivePage.goto('/');

      // Test navigation menu
      if (device === 'iPhone 12') {
        await responsivePage.click('[data-testid="mobile-menu-toggle"]');
      }
      
      await expect(responsivePage.locator('[data-testid="navigation"]')).toBeVisible();

      // Test AgriDAO grid layout
      await responsivePage.goto('/agridaos');
      const grid = responsivePage.locator('[data-testid="agridao-grid"]');
      await expect(grid).toBeVisible();

      // Verify responsive grid columns
      const gridStyle = await grid.evaluate(el => 
        window.getComputedStyle(el).gridTemplateColumns
      );
      
      if (device === 'iPhone 12') {
        expect(gridStyle).toContain('1fr'); // Single column on mobile
      } else if (device === 'iPad') {
        expect(gridStyle).toContain('repeat(2'); // Two columns on tablet
      } else {
        expect(gridStyle).toContain('repeat(3'); // Three columns on desktop
      }
    });
  });
});
```

## 🔍 Performance Testing

### Load Testing with Artillery

```yaml
# tests/performance/load-test.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
    - duration: 120
      arrivalRate: 20
    - duration: 60
      arrivalRate: 50
  payload:
    path: "./test-data.csv"
    fields:
      - "walletAddress"
      - "agriDAOId"

scenarios:
  - name: "API Load Test"
    weight: 70
    flow:
      - get:
          url: "/api/agridao"
      - think: 2
      - get:
          url: "/api/agridao/{{ agriDAOId }}"
      - think: 1
      - post:
          url: "/api/consultation"
          json:
            cropType: "rice"
            symptoms: ["yellowing", "wilting"]
            walletAddress: "{{ walletAddress }}"

  - name: "WebSocket Load Test"
    weight: 30
    engine: ws
    flow:
      - connect:
          target: "ws://localhost:3000"
      - send:
          payload: '{"type": "subscribe", "channel": "agridao-updates"}'
      - think: 30
      - send:
          payload: '{"type": "unsubscribe", "channel": "agridao-updates"}'
```

### Performance Monitoring

```javascript
// tests/performance/performance.test.js
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

describe('Performance Tests', () => {
  let chrome;
  let port;

  beforeAll(async () => {
    chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });
    port = chrome.port;
  });

  afterAll(async () => {
    await chrome.kill();
  });

  test('Homepage performance meets standards', async () => {
    const options = {
      logLevel: 'info',
      output: 'json',
      onlyCategories: ['performance'],
      port
    };

    const runnerResult = await lighthouse('http://localhost:3000', options);
    const performanceScore = runnerResult.lhr.categories.performance.score * 100;

    expect(performanceScore).toBeGreaterThan(90);
  });

  test('AgriDAO list page performance', async () => {
    const options = {
      logLevel: 'info',
      output: 'json',
      onlyCategories: ['performance'],
      port
    };

    const runnerResult = await lighthouse('http://localhost:3000/agridaos', options);
    const performanceScore = runnerResult.lhr.categories.performance.score * 100;

    expect(performanceScore).toBeGreaterThan(85);
  });
});
```

## 🔒 Security Testing

### Security Test Suite

```javascript
// tests/security/security.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('Security Tests', () => {
  describe('SQL Injection Protection', () => {
    it('should prevent SQL injection in AgriDAO search', async () => {
      const maliciousQuery = "'; DROP TABLE agri_dao; --";
      
      const response = await request(app)
        .get(`/api/agridao?search=${encodeURIComponent(maliciousQuery)}`)
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data).toEqual([]);
    });
  });

  describe('XSS Protection', () => {
    it('should sanitize HTML input in DAO creation', async () => {
      const maliciousData = {
        name: '<script>alert("xss")</script>',
        description: '<img src="x" onerror="alert(1)">',
        category: 'rice',
        minimumStake: '100',
        walletAddress: '0x1234567890123456789012345678901234567890'
      };

      const response = await request(app)
        .post('/api/agridao')
        .send(maliciousData)
        .expect(201);

      expect(response.body.data.name).not.toContain('<script>');
      expect(response.body.data.description).not.toContain('<img');
    });
  });

  describe('Rate Limiting', () => {
    it('should rate limit API requests', async () => {
      const requests = Array(101).fill().map(() => 
        request(app).get('/api/agridao')
      );

      const responses = await Promise.all(requests);
      const rateLimitedResponses = responses.filter(r => r.status === 429);

      expect(rateLimitedResponses.length).toBeGreaterThan(0);
    });
  });

  describe('Authentication Security', () => {
    it('should require valid signature for wallet authentication', async () => {
      const invalidSignature = {
        walletAddress: '0x1234567890123456789012345678901234567890',
        signature: 'invalid-signature',
        message: 'Login to Optyma'
      };

      const response = await request(app)
        .post('/api/auth/login')
        .send(invalidSignature)
        .expect(401);

      expect(response.body.error).toBe('Invalid signature');
    });
  });
});

// tests/security/smart-contract-security.test.js
describe('Smart Contract Security', () => {
  it('should prevent reentrancy attacks', async () => {
    // Test reentrancy protection in staking functions
    const maliciousContract = await ethers.getContractFactory("MaliciousReentrancy");
    const attacker = await maliciousContract.deploy(optimaToken.address);

    await expect(
      attacker.attack()
    ).to.be.revertedWith("ReentrancyGuard: reentrant call");
  });

  it('should validate access control', async () => {
    await expect(
      optimaToken.connect(user1).mint(user1.address, ethers.utils.parseEther("1000"))
    ).to.be.revertedWith("Ownable: caller is not the owner");
  });

  it('should handle integer overflow/underflow', async () => {
    const maxUint256 = ethers.constants.MaxUint256;
    
    await expect(
      optimaToken.transfer(user1.address, maxUint256.add(1))
    ).to.be.revertedWith("SafeMath: addition overflow");
  });
});
```

## 📊 Test Coverage & Reporting

### Coverage Configuration

```javascript
// jest.config.js
module.exports = {
  collectCoverageFrom: [
    'src/**/*.{js,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/contracts/**',
    '!src/**/*.config.js'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/components/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    },
    './src/hooks/': {
      branches: 85,
      functions: 85,
      lines: 85,
      statements: 85
    }
  },
  coverageReporters: ['text', 'lcov', 'html'],
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js']
};
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npx playwright install
      - run: npm run test:e2e

  security-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm audit
      - run: npm run test:security
      - uses: securecodewarrior/github-action-add-sarif@v1
        with:
          sarif-file: security-report.sarif
```

## 🎯 Test Quality Metrics

### Quality Gates

```javascript
// Test quality requirements
const QUALITY_GATES = {
  // Coverage Requirements
  unitTestCoverage: 80,
  integrationTestCoverage: 70,
  e2eTestCoverage: 60,
  
  // Performance Requirements
  maxResponseTime: 200, // ms
  maxPageLoadTime: 3000, // ms
  minLighthouseScore: 90,
  
  // Security Requirements
  maxSecurityVulnerabilities: 0,
  maxDependencyVulnerabilities: 5,
  
  // Reliability Requirements
  maxTestFailureRate: 2, // %
  minTestStability: 95 // %
};
```

### Monitoring & Alerts

```typescript
// Test monitoring dashboard
interface TestMetrics {
  coverage: {
    unit: number;
    integration: number;
    e2e: number;
  };
  performance: {
    avgResponseTime: number;
    avgPageLoad: number;
    lighthouseScore: number;
  };
  security: {
    vulnerabilities: number;
    securityScore: number;
  };
  reliability: {
    testFailureRate: number;
    flakyTests: string[];
    avgTestDuration: number;
  };
}
```

---

*Strategi testing ini memastikan kualitas tinggi dan keandalan platform Optyma Agritech Protocol di semua level - dari unit tests hingga end-to-end testing.*
