# Fetch Legends - Game Design Document

## Core Game Concept

**Fetch Legends**: Learn HTTP methods by solving puzzles as a knight adventurer

A 2D side-scrolling educational game built with Vue.js, Node.js, and TypeScript to teach developers how to use API endpoints through interactive gameplay.

---

## Game Mechanics Breakdown

### 1. Movement & Basic Interactions

- Knight sprite with left/right and up movement (arrow keys or A/D/W)
- Simple collision detection (bounding boxes, no physics)
- Interaction prompt when near objects (press E or click)

### 2. Endpoint Learning System
Each HTTP method = different action:
- **GET** - Read/Retrieve (examine signs, read scrolls, check inventory)
- **POST** - Create (craft items, open new paths, spawn objects)
- **PATCH** - Modify (upgrade items, unlock doors, change object states)
- **PUT** - Replace (swap items, transform objects)
- **DELETE** - Remove (clear obstacles, destroy enemies)

### 3. Interaction Flow
```
Player approaches object → Prompt appears with API challenge
→ Code editor opens → Player writes endpoint call
→ Game validates request → Object responds accordingly
```

---

## Level Structure Ideas

### Level 1: The Village (GET basics)
- Read village signs (GET /signs/:id)
- Check inventory (GET /inventory)
- Examine NPCs (GET /npcs/:name)

### Level 2: The Armory (POST - Creation)
- Craft sword (POST /items {type: "sword"})
- Create shield (POST /items {type: "shield"})
- Forge armor pieces

### Level 3: The Dungeon (PATCH - Modification)
- Unlock doors (PATCH /doors/:id {status: "open"})
- Upgrade weapons (PATCH /items/:id {level: 2})
- Light torches

### Level 4: The Castle (Mixed Methods)
- Combine all learned methods
- Boss fight using DELETE to defeat enemy

### Level 5: The Dragon's Lair (Advanced)
- Query parameters (GET /treasure?rarity=legendary)
- Request headers
- Error handling (404, 400, etc.)

---

## Technical Architecture

### Frontend (Vue.js + TypeScript)

```
/src
  /components
    GameCanvas.vue          # Main game renderer
    CodeEditor.vue          # Where users write endpoints
    KnightSprite.vue        # Player character
    InteractiveObject.vue   # Doors, chests, enemies
    HUD.vue                 # Health, inventory, level info
  
  /game
    /entities
      Knight.ts             # Player class
      GameObject.ts         # Base class for objects
      Enemy.ts
    
    /managers
      GameState.ts          # Current level, score, inventory
      CollisionManager.ts   # Simple AABB collision
      InputManager.ts       # Keyboard handling
    
    /levels
      LevelConfig.ts        # Level data structure
      level1.ts, level2.ts...
  
  /api
    APIService.ts           # Backend API communication
```

### Backend (Node.js + TypeScript)

```
/server
  /src
    /routes
      items.ts              # CRUD for items
      doors.ts              # Door interactions
      enemies.ts            # Enemy interactions
      inventory.ts          # Player inventory management
      npcs.ts               # NPC interactions
      player.ts             # Player state and progress
      signs.ts              # Village signs and readable objects
      
    /controllers
      ItemController.ts     # Business logic for items
      DoorController.ts     # Door state management
      EnemyController.ts    # Enemy interactions
      ChallengeController.ts # Validate player solutions
      InventoryController.ts # Inventory operations
      
    /services
      ValidationService.ts  # Check if endpoint calls are correct
      ProgressService.ts    # Track player progress
      GameStateService.ts   # Manage game state
      ChallengeService.ts   # Generate and validate challenges
      
    /models
      Item.ts               # Item data model
      Player.ts             # Player data model
      Level.ts              # Level configuration
      Challenge.ts          # Challenge data model
      GameState.ts          # Game state model
      
    /middleware
      errorHandler.ts       # Error handling middleware
      validator.ts          # Request validation
      cors.ts               # CORS configuration
      
    /config
      database.ts           # Database configuration (if needed)
      endpoints.ts          # Endpoint configurations per level
      challenges.ts         # Challenge definitions
      
    /types
      index.ts              # Shared TypeScript types
      
    server.ts               # Express app setup
    app.ts                  # Main application entry
```

---

## Backend Implementation Details

### API Endpoints by Feature

#### Items API
```typescript
// GET - Retrieve items
GET    /api/items              # Get all available items
GET    /api/items/:id          # Get specific item details

// POST - Create new items
POST   /api/items              # Create new item
Body: { type: string, name: string, stats?: object }

// PATCH - Modify items
PATCH  /api/items/:id          # Update item properties
Body: { level?: number, enchantment?: string }

// PUT - Replace items
PUT    /api/items/:id          # Replace entire item
Body: { type: string, name: string, stats: object }

// DELETE - Remove items
DELETE /api/items/:id          # Delete item
```

#### Doors API
```typescript
// GET - Check door status
GET    /api/doors/:id          # Get door state

// PATCH - Modify door state
PATCH  /api/doors/:id          # Unlock/open doors
Body: { locked: boolean, open: boolean }
```

#### Inventory API
```typescript
// GET - View inventory
GET    /api/inventory          # Get player's inventory
GET    /api/inventory/:slot    # Get specific inventory slot

// POST - Add to inventory
POST   /api/inventory          # Add item to inventory
Body: { itemId: string, quantity: number }

// DELETE - Remove from inventory
DELETE /api/inventory/:itemId  # Remove item from inventory
```

#### NPCs API
```typescript
// GET - Interact with NPCs
GET    /api/npcs               # Get all NPCs in area
GET    /api/npcs/:name         # Get specific NPC dialogue
```

#### Player API
```typescript
// GET - Player information
GET    /api/player             # Get player state
GET    /api/player/progress    # Get level progress

// PATCH - Update player state
PATCH  /api/player             # Update player properties
Body: { health?: number, position?: object }

// POST - Player actions
POST   /api/player/actions     # Perform player action
Body: { action: string, target?: string }
```

#### Enemies API
```typescript
// GET - Enemy information
GET    /api/enemies            # Get all enemies in area
GET    /api/enemies/:id        # Get specific enemy

// PATCH - Damage enemy
PATCH  /api/enemies/:id        # Update enemy health
Body: { health: number }

// DELETE - Defeat enemy
DELETE /api/enemies/:id        # Remove defeated enemy
```

### Sample Controller Implementation

```typescript
// controllers/DoorController.ts
import { Request, Response } from 'express';
import { GameStateService } from '../services/GameStateService';
import { ValidationService } from '../services/ValidationService';

export class DoorController {
  static async getDoorState(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const door = await GameStateService.getDoor(id);
      
      if (!door) {
        return res.status(404).json({
          error: 'Door not found',
          hint: 'Check the door ID and try again'
        });
      }
      
      return res.status(200).json(door);
    } catch (error) {
      return res.status(500).json({ error: 'Internal server error' });
    }
  }
  
  static async updateDoorState(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const { locked, open } = req.body;
      
      // Validate the request
      const validation = ValidationService.validateDoorUpdate(req.body);
      if (!validation.valid) {
        return res.status(400).json({
          error: 'Invalid request',
          message: validation.message,
          hint: 'Use PATCH /doors/:id with body { locked: false }'
        });
      }
      
      const updatedDoor = await GameStateService.updateDoor(id, { locked, open });
      
      return res.status(200).json({
        success: true,
        message: 'The door creaks open!',
        door: updatedDoor
      });
    } catch (error) {
      return res.status(500).json({ error: 'Internal server error' });
    }
  }
}
```

### Sample Validation Service

```typescript
// services/ValidationService.ts
interface ValidationResult {
  valid: boolean;
  message?: string;
  hints?: string[];
}

export class ValidationService {
  static validateDoorUpdate(body: any): ValidationResult {
    if (!body || typeof body !== 'object') {
      return {
        valid: false,
        message: 'Request body is required',
        hints: ['Include a JSON body with your request']
      };
    }
    
    if (body.locked === undefined && body.open === undefined) {
      return {
        valid: false,
        message: 'Must specify locked or open property',
        hints: ['Try: { locked: false } or { open: true }']
      };
    }
    
    return { valid: true };
  }
  
  static validateItemCreation(body: any): ValidationResult {
    if (!body.type) {
      return {
        valid: false,
        message: 'Item type is required',
        hints: ['Include type property: { type: "sword" }']
      };
    }
    
    const validTypes = ['sword', 'shield', 'helmet', 'armor'];
    if (!validTypes.includes(body.type)) {
      return {
        valid: false,
        message: `Invalid item type: ${body.type}`,
        hints: [`Valid types: ${validTypes.join(', ')}`]
      };
    }
    
    return { valid: true };
  }
  
  static validateEndpointCall(
    method: string,
    path: string,
    body: any,
    expected: Challenge
  ): ValidationResult {
    // Check HTTP method
    if (method !== expected.correctEndpoint.method) {
      return {
        valid: false,
        message: `Wrong HTTP method. Expected ${expected.correctEndpoint.method}`,
        hints: [expected.hint]
      };
    }
    
    // Check path
    if (path !== expected.correctEndpoint.path) {
      return {
        valid: false,
        message: 'Incorrect endpoint path',
        hints: [`Expected: ${expected.correctEndpoint.path}`]
      };
    }
    
    // Check body if required
    if (expected.correctEndpoint.body) {
      const bodyMatch = this.compareObjects(body, expected.correctEndpoint.body);
      if (!bodyMatch) {
        return {
          valid: false,
          message: 'Request body does not match expected format',
          hints: [`Expected: ${JSON.stringify(expected.correctEndpoint.body)}`]
        };
      }
    }
    
    return { valid: true };
  }
  
  private static compareObjects(obj1: any, obj2: any): boolean {
    return JSON.stringify(obj1) === JSON.stringify(obj2);
  }
}
```

### Database Schema (Optional - Can use in-memory for simplicity)

```typescript
// models/Player.ts
interface Player {
  id: string;
  name: string;
  level: number;
  currentLevel: number;
  inventory: Item[];
  health: number;
  position: { x: number; y: number };
  progress: {
    levelsCompleted: number[];
    challengesCompleted: string[];
    stars: number;
  };
}

// models/GameState.ts
interface GameState {
  playerId: string;
  currentLevel: number;
  doors: Map<string, DoorState>;
  enemies: Map<string, EnemyState>;
  items: Map<string, Item>;
  npcs: Map<string, NPCState>;
}

// models/Item.ts
interface Item {
  id: string;
  type: 'sword' | 'shield' | 'helmet' | 'armor' | 'potion';
  name: string;
  stats: {
    attack?: number;
    defense?: number;
    health?: number;
  };
  level: number;
}
```

### Express Server Setup

```typescript
// server.ts
import express from 'express';
import cors from 'cors';
import { itemRoutes } from './routes/items';
import { doorRoutes } from './routes/doors';
import { inventoryRoutes } from './routes/inventory';
import { playerRoutes } from './routes/player';
import { enemyRoutes } from './routes/enemies';
import { npcRoutes } from './routes/npcs';
import { errorHandler } from './middleware/errorHandler';

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());

// Routes
app.use('/api/items', itemRoutes);
app.use('/api/doors', doorRoutes);
app.use('/api/inventory', inventoryRoutes);
app.use('/api/player', playerRoutes);
app.use('/api/enemies', enemyRoutes);
app.use('/api/npcs', npcRoutes);

// Error handling
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(`🎮 Fetch Legends API running on port ${PORT}`);
});

export default app;
```

### Error Response Format

```typescript
// Standardized error responses for learning
interface APIError {
  error: string;           // Error message
  statusCode: number;      // HTTP status code
  hint?: string;           // Educational hint
  example?: string;        // Example of correct usage
  documentation?: string;  // Link to docs
}

// Example responses:
// 400 Bad Request
{
  error: "Invalid request body",
  statusCode: 400,
  hint: "The door requires a 'locked' property",
  example: "PATCH /doors/entrance { locked: false }"
}

// 404 Not Found
{
  error: "Resource not found",
  statusCode: 404,
  hint: "This door doesn't exist in the current level",
  example: "Try /doors/entrance or /doors/cellar"
}

// 405 Method Not Allowed
{
  error: "Method not allowed",
  statusCode: 405,
  hint: "This endpoint doesn't support GET requests",
  example: "Use PATCH /doors/:id instead"
}
```

### Challenge Validation Flow

```typescript
// When player submits an endpoint call
1. Frontend captures the request (method, path, body)
2. Frontend sends validation request to backend
3. Backend validates against expected challenge
4. Backend returns success/failure with hints
5. Frontend triggers appropriate game response

// Example validation endpoint
POST /api/challenges/validate
Body: {
  challengeId: string,
  playerRequest: {
    method: string,
    path: string,
    body?: any
  }
}

Response: {
  correct: boolean,
  message: string,
  hints?: string[],
  reward?: any
}
```

---

## Frontend-Backend Communication

### API Service Pattern

```typescript
// api/APIService.ts
import axios, { AxiosInstance } from 'axios';

class APIService {
  private client: AxiosInstance;
  
  constructor() {
    this.client = axios.create({
      baseURL: import.meta.env.VITE_API_URL || 'http://localhost:3000/api',
      headers: {
        'Content-Type': 'application/json'
      }
    });
  }
  
  // Items
  async getItems() {
    return this.client.get('/items');
  }
  
  async createItem(itemData: any) {
    return this.client.post('/items', itemData);
  }
  
  async updateItem(id: string, updates: any) {
    return this.client.patch(`/items/${id}`, updates);
  }
  
  // Doors
  async getDoorState(id: string) {
    return this.client.get(`/doors/${id}`);
  }
  
  async unlockDoor(id: string) {
    return this.client.patch(`/doors/${id}`, { locked: false });
  }
  
  // Challenge Validation
  async validateChallenge(challengeId: string, playerRequest: any) {
    return this.client.post('/challenges/validate', {
      challengeId,
      playerRequest
    });
  }
}

export default new APIService();
```

### Game Flow with Backend Integration

```
1. Player approaches interactive object (e.g., door)
   ↓
2. Frontend displays challenge prompt and code editor
   ↓
3. Player writes endpoint call:
   PATCH /doors/entrance { locked: false }
   ↓
4. Frontend sends validation request to backend:
   POST /api/challenges/validate
   Body: {
     challengeId: "unlock_door_1",
     playerRequest: {
       method: "PATCH",
       path: "/doors/entrance",
       body: { locked: false }
     }
   }
   ↓
5. Backend validates the request:
   - Checks HTTP method
   - Validates endpoint path
   - Verifies request body
   ↓
6. Backend responds:
   Success: { correct: true, message: "Door unlocked!", reward: {...} }
   Failure: { correct: false, hint: "Try using PATCH instead of GET" }
   ↓
7. Frontend processes response:
   - Success: Animate door opening, play sound, grant reward
   - Failure: Show error message with hint, let player retry
```

### Error Handling in Frontend

```typescript
// Example error handling in Vue component
async function submitEndpoint(method: string, path: string, body?: any) {
  try {
    const result = await APIService.validateChallenge(currentChallenge.value.id, {
      method,
      path,
      body
    });
    
    if (result.data.correct) {
      // Success!
      showSuccessAnimation();
      updateGameState(result.data.reward);
      playSound('success');
    } else {
      // Show helpful error
      showError(result.data.hint);
      incorrectAttempts++;
    }
  } catch (error) {
    if (error.response) {
      // Backend returned an error
      showError(error.response.data.hint || 'Something went wrong');
    } else {
      // Network error
      showError('Cannot connect to server');
    }
  }
}
```

---

## Simplified Game Loop

```typescript
// Pseudo-code structure
class Game {
  knight: Knight
  currentLevel: Level
  objects: GameObject[]
  
  update(deltaTime: number) {
    // 1. Handle input
    this.knight.move(input)
    
    // 2. Check collisions
    const nearbyObject = this.checkProximity()
    
    // 3. Show interaction prompt
    if (nearbyObject) {
      this.showPrompt(nearbyObject.challenge)
    }
    
    // 4. Render
    this.render()
  }
}
```

---

## Key Features to Keep It Simple

✅ **Canvas-based rendering** (or SVG for simplicity)

✅ **Grid-based movement** (no need for smooth physics)

✅ **Pre-defined object positions** in each level

✅ **Real backend API** with educational error messages

✅ **Visual feedback** for correct/incorrect requests

✅ **Progressive difficulty** (start with just GET, add methods gradually)

---

## Sample Challenge Format

```typescript
interface Challenge {
  description: "Open the wooden door"
  hint: "Use PATCH to change the door's state"
  correctEndpoint: {
    method: "PATCH",
    path: "/doors/entrance",
    body: { locked: false }
  }
  successMessage: "The door creaks open!"
  reward: "key_item" | "experience" | "next_area"
}
```

---

## Progression System (none for now)

---

## Visual Style Suggestions

- **Pixel art** (8-bit or 16-bit style)
- **Limited color palette** (easier to create assets)
- **Tile-based backgrounds** (reusable patterns)
- **Simple animations** (2-3 frames per action)

---

## Development Phases

### Phase 1: Core Mechanics (MVP)
- [ ] Basic knight movement
- [ ] Simple collision detection
- [ ] One interactive object (door)
- [ ] Code editor component
- [ ] **Backend: Express server setup**
- [ ] **Backend: Door API endpoint (GET, PATCH)**
- [ ] **Backend: Challenge validation service**
- [ ] GET endpoint validation
- [ ] One complete level

### Phase 2: Content Expansion
- [ ] Add POST, PATCH, DELETE methods
- [ ] **Backend: Items API (full CRUD)**
- [ ] **Backend: Inventory API**
- [ ] **Backend: Enemies API**
- [ ] **Backend: NPCs API**
- [ ] Create 3-5 levels
- [ ] Add inventory system
- [ ] Implement progression tracking

### Phase 3: Polish
- [ ] Add animations
- [ ] Sound effects
- [ ] Tutorial system
- [ ] Achievement system
- [ ] Level selector UI

### Phase 4: Advanced Features
- [ ] Query parameters
- [ ] Request headers
- [ ] Error handling scenarios
- [ ] Boss battles
- [ ] Multiplayer/leaderboards (optional)

---

## Technical Considerations

### State Management
- Use Vue's Composition API for game state
- Pinia for global state (inventory, progress)
- LocalStorage for saving progress

### Rendering Options

1. **HTML5 Canvas** - Better for sprite rendering

### API Implementation

- Real Node.js/Express backend
- RESTful API design
- Validate endpoint structure (method, path, body)
- Provide helpful, educational error messages
- Support for all HTTP methods (GET, POST, PATCH, PUT, DELETE)

---

## Example Level Data Structure

```typescript
interface Level {
  id: number
  name: string
  description: string
  background: string
  objects: GameObject[]
  startPosition: { x: number, y: number }
  objectives: Objective[]
  stars: {
    time: number
    accuracy: number
    style: boolean
  }
}

interface GameObject {
  id: string
  type: 'door' | 'chest' | 'npc' | 'enemy'
  position: { x: number, y: number }
  sprite: string
  challenge?: Challenge
  state: any
}
```

---

## Success Metrics

- Players understand HTTP methods
- Can construct basic API requests
- Complete levels with minimal hints
- Positive feedback on learning experience
- Low frustration rate

---

## Future Expansion Ideas

- WebSocket level (real-time connections)
- Authentication level (JWT tokens)
- REST vs GraphQL comparison level
- API versioning scenarios
- Rate limiting challenges

---

## Running the Project

### Backend Setup
```bash
cd server
npm install
npm run dev
# Server runs on http://localhost:3000
```

### Frontend Setup
```bash
cd client
npm install
npm run dev
# Client runs on http://localhost:5173 (Vite default)
```

### Environment Variables
```bash
# server/.env
PORT=3000
NODE_ENV=development
CORS_ORIGIN=http://localhost:5173

# client/.env
VITE_API_URL=http://localhost:3000/api
```

---

## Notes

- Keep it lightweight (no heavy frameworks beyond Vue)
- Prioritize learning over complex gameplay
- Clear feedback on what went wrong
- Make it fun but educational
- Mobile-friendly responsive design
