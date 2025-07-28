# OPTYMA AGRITECH PROTOCOL - Smart Contract Architecture

## 🏗️ Arsitektur Smart Contract

### Contract Hierarchy

```
OptimaProtocol (Proxy)
├── OptimaToken.sol (ERC-20 Governance Token)
├── AgriDAOFactory.sol (DAO Creation & Management)
├── KnowledgeTokens.sol (ERC-721 Knowledge NFTs)
├── AntiScamRegistry.sol (Supplier Verification)
├── StakingRewards.sol (Staking & Reward Distribution)
├── Treasury.sol (Protocol Treasury Management)
└── Governance.sol (Voting & Proposals)
```

## 📋 Detail Smart Contracts

### 1. OptimaToken.sol
**Fungsi**: Governance dan utility token untuk ecosystem

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract OptimaToken is ERC20, Ownable {
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 10**18; // 1B tokens
    
    // Tokenomics Distribution
    uint256 public constant COMMUNITY_ALLOCATION = TOTAL_SUPPLY * 40 / 100; // 40%
    uint256 public constant TEAM_ALLOCATION = TOTAL_SUPPLY * 20 / 100;      // 20%
    uint256 public constant TREASURY_ALLOCATION = TOTAL_SUPPLY * 25 / 100;   // 25%
    uint256 public constant LIQUIDITY_ALLOCATION = TOTAL_SUPPLY * 15 / 100;  // 15%
    
    mapping(address => bool) public stakingContracts;
    
    constructor() ERC20("Optyma Protocol", "OPTYMA") {
        _mint(msg.sender, TOTAL_SUPPLY);
    }
    
    function addStakingContract(address _contract) external onlyOwner {
        stakingContracts[_contract] = true;
    }
    
    // Voting power calculation
    function getVotingPower(address account) external view returns (uint256) {
        return balanceOf(account) + getStakedBalance(account);
    }
    
    function getStakedBalance(address account) public view returns (uint256) {
        // Implementation for getting staked balance across all staking contracts
    }
}
```

**Features**:
- Total Supply: 1 Miliar OPTYMA tokens
- Voting power termasuk staked tokens
- Integration dengan staking contracts
- Vesting schedule untuk team allocation

### 2. AgriDAOFactory.sol
**Fungsi**: Factory contract untuk membuat dan mengelola AgriDAOs

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "./AgriDAO.sol";
import "./interfaces/IOptimaProtocol.sol";

contract AgriDAOFactory {
    IOptimaProtocol public immutable protocol;
    
    struct DAOInfo {
        address daoAddress;
        string specialty;      // "aquaculture", "livestock", "crops", etc.
        address[] consultants;
        uint256 createdAt;
        bool isActive;
    }
    
    mapping(uint256 => DAOInfo) public daos;
    mapping(string => uint256[]) public specialtyToDAOs;
    uint256 public daoCounter;
    
    event DAOCreated(uint256 indexed daoId, address daoAddress, string specialty);
    
    constructor(address _protocol) {
        protocol = IOptimaProtocol(_protocol);
    }
    
    function createAgriDAO(
        string memory _name,
        string memory _symbol,
        string memory _specialty,
        address[] memory _initialConsultants,
        uint256 _targetRaise
    ) external returns (uint256 daoId) {
        require(protocol.canCreateDAO(msg.sender), "Not authorized");
        
        daoId = daoCounter++;
        
        AgriDAO newDAO = new AgriDAO(
            _name,
            _symbol,
            _specialty,
            _initialConsultants,
            _targetRaise,
            address(this)
        );
        
        daos[daoId] = DAOInfo({
            daoAddress: address(newDAO),
            specialty: _specialty,
            consultants: _initialConsultants,
            createdAt: block.timestamp,
            isActive: true
        });
        
        specialtyToDAOs[_specialty].push(daoId);
        
        emit DAOCreated(daoId, address(newDAO), _specialty);
        
        return daoId;
    }
    
    function getDAOsBySpecialty(string memory _specialty) 
        external view returns (uint256[] memory) {
        return specialtyToDAOs[_specialty];
    }
}
```

### 3. AgriDAO.sol
**Fungsi**: Individual DAO contract untuk setiap spesialisasi

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract AgriDAO is ERC20, ReentrancyGuard {
    struct Consultant {
        address wallet;
        string expertise;
        uint256 yearsExperience;
        bool isVerified;
        uint256 consultationFee;
    }
    
    struct Project {
        uint256 id;
        string title;
        string description;
        uint256 fundingGoal;
        uint256 currentFunding;
        address proposer;
        bool isActive;
        uint256 deadline;
    }
    
    mapping(address => Consultant) public consultants;
    mapping(uint256 => Project) public projects;
    
    string public specialty;
    uint256 public targetRaise;
    uint256 public currentRaise;
    address public factory;
    
    uint256 public projectCounter;
    uint256 public consultationFeePool;
    
    event ConsultantAdded(address indexed consultant, string expertise);
    event ProjectProposed(uint256 indexed projectId, address proposer);
    event ProjectFunded(uint256 indexed projectId, address funder, uint256 amount);
    event ConsultationBooked(address indexed consultant, address client, uint256 fee);
    
    constructor(
        string memory _name,
        string memory _symbol,
        string memory _specialty,
        address[] memory _initialConsultants,
        uint256 _targetRaise,
        address _factory
    ) ERC20(_name, _symbol) {
        specialty = _specialty;
        targetRaise = _targetRaise;
        factory = _factory;
        
        // Add initial consultants
        for (uint i = 0; i < _initialConsultants.length; i++) {
            consultants[_initialConsultants[i]].wallet = _initialConsultants[i];
            consultants[_initialConsultants[i]].isVerified = true;
        }
    }
    
    function proposeProject(
        string memory _title,
        string memory _description,
        uint256 _fundingGoal,
        uint256 _deadline
    ) external returns (uint256) {
        uint256 projectId = projectCounter++;
        
        projects[projectId] = Project({
            id: projectId,
            title: _title,
            description: _description,
            fundingGoal: _fundingGoal,
            currentFunding: 0,
            proposer: msg.sender,
            isActive: true,
            deadline: _deadline
        });
        
        emit ProjectProposed(projectId, msg.sender);
        return projectId;
    }
    
    function fundProject(uint256 _projectId) external payable nonReentrant {
        Project storage project = projects[_projectId];
        require(project.isActive, "Project not active");
        require(block.timestamp < project.deadline, "Project deadline passed");
        
        project.currentFunding += msg.value;
        
        // Mint DAO tokens proportional to funding
        uint256 tokensToMint = (msg.value * 1000) / 1 ether; // 1000 tokens per ETH
        _mint(msg.sender, tokensToMint);
        
        emit ProjectFunded(_projectId, msg.sender, msg.value);
    }
    
    function bookConsultation(address _consultant) external payable nonReentrant {
        Consultant storage consultant = consultants[_consultant];
        require(consultant.isVerified, "Consultant not verified");
        require(msg.value >= consultant.consultationFee, "Insufficient fee");
        
        // 90% to consultant, 10% to DAO treasury
        uint256 consultantFee = (msg.value * 90) / 100;
        uint256 daoFee = msg.value - consultantFee;
        
        payable(_consultant).transfer(consultantFee);
        consultationFeePool += daoFee;
        
        emit ConsultationBooked(_consultant, msg.sender, msg.value);
    }
}
```

### 4. KnowledgeTokens.sol
**Fungsi**: NFT contract untuk tokenisasi pengetahuan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract KnowledgeTokens is ERC721 {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;
    
    enum KnowledgeType { GUIDE, CONSULTATION, DATA, EXPERIENCE }
    
    struct KnowledgeToken {
        uint256 id;
        KnowledgeType tokenType;
        string title;
        string description;
        string ipfsHash;        // Content stored on IPFS
        address creator;
        uint256 price;
        uint256 accessCount;
        bool isForSale;
    }
    
    mapping(uint256 => KnowledgeToken) public knowledgeTokens;
    mapping(address => uint256[]) public creatorTokens;
    mapping(KnowledgeType => uint256[]) public tokensByType;
    
    uint256 public platformFeePercent = 250; // 2.5%
    address public treasury;
    
    event KnowledgeTokenMinted(
        uint256 indexed tokenId, 
        address indexed creator, 
        KnowledgeType tokenType
    );
    
    event KnowledgeTokenSold(
        uint256 indexed tokenId,
        address indexed seller,
        address indexed buyer,
        uint256 price
    );
    
    constructor(address _treasury) ERC721("Optyma Knowledge Tokens", "OKT") {
        treasury = _treasury;
    }
    
    function mintKnowledgeToken(
        KnowledgeType _type,
        string memory _title,
        string memory _description,
        string memory _ipfsHash,
        uint256 _price
    ) external returns (uint256) {
        _tokenIds.increment();
        uint256 newTokenId = _tokenIds.current();
        
        _mint(msg.sender, newTokenId);
        
        knowledgeTokens[newTokenId] = KnowledgeToken({
            id: newTokenId,
            tokenType: _type,
            title: _title,
            description: _description,
            ipfsHash: _ipfsHash,
            creator: msg.sender,
            price: _price,
            accessCount: 0,
            isForSale: _price > 0
        });
        
        creatorTokens[msg.sender].push(newTokenId);
        tokensByType[_type].push(newTokenId);
        
        emit KnowledgeTokenMinted(newTokenId, msg.sender, _type);
        
        return newTokenId;
    }
    
    function purchaseKnowledgeAccess(uint256 _tokenId) external payable {
        KnowledgeToken storage token = knowledgeTokens[_tokenId];
        require(token.isForSale, "Token not for sale");
        require(msg.value >= token.price, "Insufficient payment");
        
        // Calculate fees
        uint256 platformFee = (msg.value * platformFeePercent) / 10000;
        uint256 creatorPayment = msg.value - platformFee;
        
        // Transfer payments
        payable(token.creator).transfer(creatorPayment);
        payable(treasury).transfer(platformFee);
        
        // Update access count
        token.accessCount++;
        
        emit KnowledgeTokenSold(_tokenId, token.creator, msg.sender, msg.value);
    }
    
    function getTokensByCreator(address _creator) 
        external view returns (uint256[] memory) {
        return creatorTokens[_creator];
    }
    
    function getTokensByType(KnowledgeType _type) 
        external view returns (uint256[] memory) {
        return tokensByType[_type];
    }
}
```

### 5. AntiScamRegistry.sol
**Fungsi**: Registry untuk verifikasi supplier dan anti-scam

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract AntiScamRegistry is AccessControl {
    bytes32 public constant VERIFIER_ROLE = keccak256("VERIFIER_ROLE");
    
    enum SupplierStatus { UNVERIFIED, VERIFIED, BLACKLISTED }
    
    struct Supplier {
        address wallet;
        string name;
        string businessLicense;
        string[] certifications;
        SupplierStatus status;
        uint256 verifiedAt;
        uint256 ratingSum;
        uint256 ratingCount;
        address verifier;
    }
    
    struct Review {
        address reviewer;
        uint256 rating;      // 1-5 stars
        string comment;
        uint256 timestamp;
        bool isVerified;     // Verified purchase
    }
    
    mapping(address => Supplier) public suppliers;
    mapping(address => Review[]) public supplierReviews;
    mapping(address => mapping(address => bool)) public hasReviewed;
    
    address[] public verifiedSuppliers;
    address[] public blacklistedSuppliers;
    
    event SupplierRegistered(address indexed supplier, string name);
    event SupplierVerified(address indexed supplier, address indexed verifier);
    event SupplierBlacklisted(address indexed supplier, string reason);
    event ReviewSubmitted(address indexed supplier, address indexed reviewer, uint256 rating);
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(VERIFIER_ROLE, msg.sender);
    }
    
    function registerSupplier(
        string memory _name,
        string memory _businessLicense,
        string[] memory _certifications
    ) external {
        require(suppliers[msg.sender].wallet == address(0), "Already registered");
        
        suppliers[msg.sender] = Supplier({
            wallet: msg.sender,
            name: _name,
            businessLicense: _businessLicense,
            certifications: _certifications,
            status: SupplierStatus.UNVERIFIED,
            verifiedAt: 0,
            ratingSum: 0,
            ratingCount: 0,
            verifier: address(0)
        });
        
        emit SupplierRegistered(msg.sender, _name);
    }
    
    function verifySupplier(address _supplier) external onlyRole(VERIFIER_ROLE) {
        Supplier storage supplier = suppliers[_supplier];
        require(supplier.wallet != address(0), "Supplier not registered");
        require(supplier.status == SupplierStatus.UNVERIFIED, "Already processed");
        
        supplier.status = SupplierStatus.VERIFIED;
        supplier.verifiedAt = block.timestamp;
        supplier.verifier = msg.sender;
        
        verifiedSuppliers.push(_supplier);
        
        emit SupplierVerified(_supplier, msg.sender);
    }
    
    function blacklistSupplier(address _supplier, string memory _reason) 
        external onlyRole(VERIFIER_ROLE) {
        Supplier storage supplier = suppliers[_supplier];
        require(supplier.wallet != address(0), "Supplier not registered");
        
        supplier.status = SupplierStatus.BLACKLISTED;
        blacklistedSuppliers.push(_supplier);
        
        emit SupplierBlacklisted(_supplier, _reason);
    }
    
    function submitReview(
        address _supplier,
        uint256 _rating,
        string memory _comment,
        bool _isVerifiedPurchase
    ) external {
        require(_rating >= 1 && _rating <= 5, "Invalid rating");
        require(!hasReviewed[_supplier][msg.sender], "Already reviewed");
        require(suppliers[_supplier].wallet != address(0), "Supplier not registered");
        
        supplierReviews[_supplier].push(Review({
            reviewer: msg.sender,
            rating: _rating,
            comment: _comment,
            timestamp: block.timestamp,
            isVerified: _isVerifiedPurchase
        }));
        
        // Update supplier rating
        Supplier storage supplier = suppliers[_supplier];
        supplier.ratingSum += _rating;
        supplier.ratingCount++;
        
        hasReviewed[_supplier][msg.sender] = true;
        
        emit ReviewSubmitted(_supplier, msg.sender, _rating);
    }
    
    function getSupplierRating(address _supplier) 
        external view returns (uint256 averageRating, uint256 totalReviews) {
        Supplier storage supplier = suppliers[_supplier];
        if (supplier.ratingCount == 0) {
            return (0, 0);
        }
        return (supplier.ratingSum / supplier.ratingCount, supplier.ratingCount);
    }
    
    function getVerifiedSuppliers() external view returns (address[] memory) {
        return verifiedSuppliers;
    }
    
    function getBlacklistedSuppliers() external view returns (address[] memory) {
        return blacklistedSuppliers;
    }
}
```

## 🔧 Deployment Strategy

### 1. Development Environment
```bash
# Install dependencies
npm install hardhat @openzeppelin/contracts

# Compile contracts
npx hardhat compile

# Run tests
npx hardhat test

# Deploy to testnet
npx hardhat run scripts/deploy.js --network mumbai
```

### 2. Contract Verification
```javascript
// hardhat.config.js
module.exports = {
  networks: {
    mumbai: {
      url: "https://rpc-mumbai.maticvigil.com/",
      accounts: [process.env.PRIVATE_KEY]
    },
    polygon: {
      url: "https://polygon-rpc.com/",
      accounts: [process.env.PRIVATE_KEY]
    }
  },
  etherscan: {
    apiKey: process.env.POLYGONSCAN_API_KEY
  }
};
```

### 3. Security Checklist
- [ ] Multi-signature wallet untuk admin functions
- [ ] Timelock untuk contract upgrades
- [ ] External audit dari ConsenSys Diligence
- [ ] Bug bounty program dengan ImmuneFi
- [ ] Formal verification untuk critical functions

### 4. Gas Optimization
- Menggunakan `uint256` untuk loop counters
- Batch operations untuk multiple transactions
- Implement EIP-2535 (Diamonds) untuk upgradeability
- Optimize storage layout untuk minimize SSTORE operations

---

*Dokumen ini mencakup smart contract architecture. Untuk deployment scripts dan testing strategies, lihat dokumen terpisah.*
