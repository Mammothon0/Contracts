# Web3ite Contract

This contract demonstrates a DApp that enables **"HTML page creation, modification, and fee management on-chain (blockchain)"**.

Users can set page ownership to one of three types:

- **Single (Individual ownership)**
- **MultiSig (Multi-signature ownership)**
- **Permissionless (Anyone can modify)**

Each page has an **Update Fee** that must be paid when requesting modifications. How these fees are stored and withdrawn varies depending on the page type.

## Core Features and Operation

### 1. Page Creation (createPage)

#### Function Signature:

```solidity
function createPage(
    string calldata _name,
    string calldata _thumbnail,
    string calldata _initialHtml,
    OwnershipConfig calldata _ownerConfig,
    uint256 _updateFee,
    bool _imt
) external returns (uint256 pageId);
```

- Registers a new HTML page on-chain
- `_name`: Page name
- `_thumbnail`: Base64 encoded thumbnail image
- `_initialHtml`: Initial HTML content (must start with DOCTYPE and end with </html>)
- `_ownerConfig`: Ownership configuration struct containing:
  - `ownershipType`: Type of ownership (Single/MultiSig/Permissionless)
  - `multiSigOwners`: List of owners for Single/MultiSig pages
  - `multiSigThreshold`: Required approvals for MultiSig
- `_updateFee`: Required payment for page modification requests (in Wei)
- `_imt`: Immutable flag for the page
- Returns `pageId`: Page identification number (increments from 1)

### 2. Update Request (requestUpdate)

#### Function Signature:

```solidity
function requestUpdate(
    uint256 _pageId,
    string calldata _newName,
    string calldata _newThumbnail,
    string calldata _newHtml
) external payable;
```

- Called when attempting to modify a page with `_pageId`
- Allows selective updates to page properties:
  - `_newName`: New page name (empty string if no change)
  - `_newThumbnail`: New thumbnail (empty string if no change)
  - `_newHtml`: New HTML content (empty string if no change)
- At least one of the three properties must be updated
- `msg.value` must be greater than or equal to the page's `Update Fee`
- Behavior varies by page type:
  - **Permissionless**: Updates are applied immediately. Update fees accumulate in `pageBalances[pageId]`
  - **Single / MultiSig**: Update requests enter a "Queue" state, requiring subsequent approval

### 3. Approval (approveRequest)

#### Function Signature:

```solidity
function approveRequest(uint256 _pageId, uint256 _requestId) external;
```

- For non-Permissionless pages (`Single/MultiSig`), owners approve pending update requests
  - **Single**: Executes immediately upon owner approval
  - **MultiSig**: Executes when `threshold` number of owners approve
- Upon execution, emits `UpdateExecuted` event and updates page HTML to `_newHtml`

### 4. Fee Withdrawal (withdrawPageFees)

#### Function Signature:

```solidity
function withdrawPageFees(uint256 _pageId) external;
```

- **Single Page**: Owner (`singleOwner`) can withdraw all accumulated fees
- **MultiSig Page**: Any listed owner can trigger equal distribution of fees among all MultiSig owners (remainder stays in contract)
- **Permissionless Page**: Withdrawal not allowed (`revert`). Instead, fees can be randomly distributed to update participants via `distributePageTreasury`

### 5. Ownership Change (changeOwnership)

#### Function Signature:

```solidity
function changeOwnership(
    uint256 _pageId,
    OwnershipType _newOwnershipType,
    address[] calldata _newMultiSigOwners,
    uint256 _newMultiSigThreshold
) external;
```

- Ownership type can only be changed from **Single** pages (`MultiSig` and `Permissionless` states cannot be changed)
- When ownership type changes, existing owner information is reset and replaced with new values
  - Example: `Single → MultiSig`, `Single → Permissionless` possible

### 6. (Optional) Permissionless Page Treasury Distribution (distributePageTreasury)

#### Function Signature:

```solidity
function distributePageTreasury(uint256 _pageId) external;
```

- Example logic that distributes accumulated fees (`pageBalances[_pageId]`) from a Permissionless page to one random participant who previously submitted update requests
- Uses block hash-based randomness, which has security limitations for mainnet services (recommend alternatives like `Chainlink VRF`)

### 7. Page Voting System (vote)

#### Function Signature:

```solidity
function vote(uint256 _pageId, bool _isLike) external;
```

- Allows users to like or dislike pages
- Features:
  - Each address can vote only once per page
  - Users can change their vote (like → dislike or vice versa)
  - Previous vote is automatically canceled when changing vote
  - Tracks total likes and dislikes per page
- Emits `VoteChanged` event with updated vote counts

---

## Contract Structure

### `IWeb3ite.sol`
- Interface defining main function/event signatures

### `Web3ite.sol`
- Contract implementing the interface
- Uses `Page` struct to store page states (`currentHtml`, `ownershipType`, `updateFee`, MultiSig info, etc.)
- Flow: `requestUpdate` → `approveRequest` → `_executeUpdate`
- Handles fee withdrawal, ownership changes via `withdrawPageFees`, `changeOwnership`, etc.

---

### Simple Usage Example

### 1. Create Page (Single Type Example)

```solidity
IWeb3ite.OwnershipConfig memory config = IWeb3ite.OwnershipConfig({
    ownershipType: IWeb3ite.OwnershipType.Single,
    multiSigOwners: new address[](1),
    multiSigThreshold: 1
});
config.multiSigOwners[0] = msg.sender;  // Set single owner

uint256 pageId = web3ite.createPage(
    "My First Page",
    "base64EncodedThumbnail",
    "<!DOCTYPE html><html>Hello On-Chain HTML!</html>",
    config,
    1e15,             // Example: 0.001 ETH as update fee
    false             // Not immutable
);
```

### 2. Request Update

```solidity
// Update only HTML content
web3ite.requestUpdate{value: 1e15}(
    pageId,
    "",                  // No name change
    "",                  // No thumbnail change
    "<h1>Updated HTML</h1>"
);

// Update name and thumbnail
web3ite.requestUpdate{value: 1e15}(
    pageId,
    "New Page Name",
    "base64EncodedNewThumbnail",
    ""                   // No HTML change
);

// Update all properties
web3ite.requestUpdate{value: 1e15}(
    pageId,
    "New Page Name",
    "base64EncodedNewThumbnail",
    "<h1>Updated HTML</h1>"
);
```

### 3. Approve (Single)

```solidity
// singleOwner approves for immediate execution
web3ite.approveRequest(pageId, 0);
```

### 4. Withdraw Fees

```solidity
// singleOwner withdraws all fees
web3ite.withdrawPageFees(pageId);
```

### 5. Vote on Page

```solidity
// Like a page
web3ite.vote(pageId, true);

// Dislike a page
web3ite.vote(pageId, false);
```

## Data Structures

### Page Struct
```solidity
struct Page {
    address[] multiSigOwners;
    string name;
    string thumbnail;
    string currentHtml;
    OwnershipType ownershipType;
    bool imt;
    uint120 totalLikes;     // Vote tracking
    uint120 totalDislikes;  // Vote tracking
    uint256 multiSigThreshold;
    uint256 updateRequestCount;
    uint256 updateFee;
    mapping(uint256 => UpdateRequest) updateRequests;
    mapping(address => bool) hasLiked;    // Vote status tracking
    mapping(address => bool) hasDisliked; // Vote status tracking
}
```

## Future Development Plans / Improvements

### HTML Storage Issue
Currently, the demo version stores HTML content as a whole, but future updates will implement element-wise storage and management. This will enable Permissionless modifications for specific sections while requiring Page Owner approval for critical parts.

### Page Ownership (Not Top Priority)
Like MillionDollar Page, specific pages can generate revenue. Plans include tokenizing page ownership as NFTs for transferability and marketplace trading.

### Domain Integration
Considering the following approaches for direct page access:
- ENS/DNS integration
- DA Layer API design for blockchain data access

### MultiSig Enhancement
Plans to integrate proven MultiSig solutions like Gnosis Safe:
- Safe module integration
- Dynamic threshold adjustment
- Enhanced security features

### Permissionless Page Random Distribution Security
Current `distributePageTreasury` uses pseudo-random generation. Planned improvements:
- Chainlink VRF integration
- Commit-Reveal scheme
- Threshold Signature-based randomness

### Permissionless Page Logic
To prevent abuse in Permissionless pages, considering:
- Staking-based modification rights
- Community voting system
- Contribution-weighted permissions
- AI-based content filtering