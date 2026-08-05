---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "configure user management with task tracking"
  - "create admin dashboard with AI insights"
  - "build user management system with anomaly detection"
  - "integrate ML service for burnout prediction"
  - "deploy enterprise user management with FastAPI"
  - "set up kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system that combines user administration, task tracking, and support ticket management with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

This system provides:
- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Management**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-based classification and routing system
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

**Stack**: React.js frontend, Node.js/Express backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `backend/.env`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:
```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `ml-service/.env`:
```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Backend)

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:userId
{
  "name": "Updated Name",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management (Backend)

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress" // or "done"
}

// Track time
POST /api/tasks/:taskId/time
{
  "duration": 3600 // seconds
}
```

### Support Tickets (Backend)

```javascript
// Create ticket
POST /api/tickets
{
  "subject": "Login issues",
  "description": "Cannot access dashboard",
  "priority": "high"
}

// Get tickets
GET /api/tickets?status=open&priority=high
```

### AI Analytics (ML Service)

```python
# Classify ticket (auto-route to department)
POST /api/ml/classify-ticket
{
  "subject": "Password reset not working",
  "description": "User cannot reset password via email"
}
# Returns: { "category": "technical", "priority": "high", "department": "IT" }

# Detect risk
POST /api/ml/risk-detection
{
  "userId": "user123",
  "behavior": {
    "login_failures": 5,
    "unusual_hours": true,
    "data_access_spike": false
  }
}
# Returns: { "risk_score": 0.75, "risk_level": "high", "factors": [...] }

# Burnout analysis
POST /api/ml/burnout-analysis
{
  "userId": "user123",
  "metrics": {
    "hours_worked_week": 65,
    "tasks_completed": 3,
    "tasks_overdue": 8,
    "weekend_work": true
  }
}
# Returns: { "burnout_score": 0.82, "recommendation": "Reduce workload" }

# Predict project delay
POST /api/ml/predict-delay
{
  "projectId": "proj123",
  "tasks_total": 50,
  "tasks_completed": 15,
  "days_elapsed": 30,
  "days_remaining": 30
}
# Returns: { "delay_probability": 0.68, "estimated_delay_days": 12 }
```

## Frontend Integration Patterns

### Authentication Hook (React)

```javascript
// frontend/src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/api/auth/login`, { email, password });
    const { token, user } = res.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
    delete axios.defaults.headers.common['Authorization'];
  };

  return { user, login, logout, isAuthenticated: !!user, isAdmin: user?.role === 'admin' };
};
```

### Task Management Component

```javascript
// frontend/src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const res = await axios.get(`${API_URL}/api/tasks`);
    const grouped = {
      todo: res.data.filter(t => t.status === 'todo'),
      inProgress: res.data.filter(t => t.status === 'in-progress'),
      done: res.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, { status: newStatus });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status === 'inProgress' ? 'In Progress' : status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <TaskCard key={task._id} task={task} onMove={moveTask} />
          ))}
        </div>
      ))}
    </div>
  );
};

const TaskCard = ({ task, onMove }) => (
  <div className="task-card">
    <h4>{task.title}</h4>
    <p>{task.description}</p>
    <div className="task-actions">
      {task.status !== 'done' && (
        <button onClick={() => onMove(task._id, task.status === 'todo' ? 'in-progress' : 'done')}>
          Move →
        </button>
      )}
    </div>
  </div>
);

export default KanbanBoard;
```

### AI Analytics Integration

```javascript
// frontend/src/components/AIInsights.jsx
import React, { useState } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIInsights = ({ userId }) => {
  const [burnoutAnalysis, setBurnoutAnalysis] = useState(null);
  const [loading, setLoading] = useState(false);

  const analyzeBurnout = async () => {
    setLoading(true);
    try {
      const res = await axios.post(`${ML_API_URL}/api/ml/burnout-analysis`, {
        userId,
        metrics: {
          hours_worked_week: 60,
          tasks_completed: 5,
          tasks_overdue: 10,
          weekend_work: true
        }
      });
      setBurnoutAnalysis(res.data);
    } catch (error) {
      console.error('Burnout analysis failed:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="ai-insights">
      <button onClick={analyzeBurnout} disabled={loading}>
        {loading ? 'Analyzing...' : 'Analyze Burnout Risk'}
      </button>
      {burnoutAnalysis && (
        <div className={`alert ${burnoutAnalysis.burnout_score > 0.7 ? 'danger' : 'warning'}`}>
          <h4>Burnout Score: {(burnoutAnalysis.burnout_score * 100).toFixed(0)}%</h4>
          <p>{burnoutAnalysis.recommendation}</p>
        </div>
      )}
    </div>
  );
};

export default AIInsights;
```

## Backend Implementation Patterns

### User Model (MongoDB/Mongoose)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  metadata: {
    loginFailures: { type: Number, default: 0 },
    suspiciousActivity: [{ timestamp: Date, action: String }]
  }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    
    if (!req.user || req.user.status !== 'active') {
      return res.status(401).json({ message: 'User not authorized' });
    }

    next();
  } catch (error) {
    res.status(401).json({ message: 'Token invalid or expired' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { protect, adminOnly };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user._id
    });
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.taskId,
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    
    // Trigger AI prediction if task completed
    if (req.body.status === 'done') {
      const project = await task.populate('projectId');
      axios.post(`${process.env.ML_SERVICE_URL}/api/ml/predict-delay`, {
        projectId: project._id,
        tasks_total: project.totalTasks,
        tasks_completed: project.completedTasks + 1
      }).catch(err => console.error('ML prediction failed:', err));
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Ticket Classification Model

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import pickle
import os

class TicketClassifier:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.vectorizer = TfidfVectorizer(max_features=1000)
        self.classifier = MultinomialNB()
        self.categories = ['technical', 'billing', 'hr', 'general']
        self.load_or_train()
    
    def load_or_train(self):
        model_file = os.path.join(self.model_path, 'ticket_classifier.pkl')
        if os.path.exists(model_file):
            with open(model_file, 'rb') as f:
                data = pickle.load(f)
                self.vectorizer = data['vectorizer']
                self.classifier = data['classifier']
        else:
            # Train with initial data if no model exists
            self.train_initial()
    
    def train_initial(self):
        # Sample training data
        texts = [
            "password reset login issue",
            "invoice payment billing",
            "vacation leave request",
            "general inquiry question"
        ]
        labels = ['technical', 'billing', 'hr', 'general']
        
        X = self.vectorizer.fit_transform(texts)
        self.classifier.fit(X, labels)
        self.save_model()
    
    def predict(self, text):
        X = self.vectorizer.transform([text])
        category = self.classifier.predict(X)[0]
        probabilities = self.classifier.predict_proba(X)[0]
        confidence = max(probabilities)
        
        return {
            'category': category,
            'confidence': float(confidence),
            'department': self.map_to_department(category)
        }
    
    def map_to_department(self, category):
        mapping = {
            'technical': 'IT',
            'billing': 'Finance',
            'hr': 'HR',
            'general': 'Support'
        }
        return mapping.get(category, 'Support')
    
    def save_model(self):
        os.makedirs(self.model_path, exist_ok=True)
        with open(os.path.join(self.model_path, 'ticket_classifier.pkl'), 'wb') as f:
            pickle.dump({
                'vectorizer': self.vectorizer,
                'classifier': self.classifier
            }, f)
```

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from models.ticket_classifier import TicketClassifier
from models.risk_detector import RiskDetector
from models.burnout_analyzer import BurnoutAnalyzer
import os

app = FastAPI(title="Enterprise ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
ticket_classifier = TicketClassifier()
risk_detector = RiskDetector()
burnout_analyzer = BurnoutAnalyzer()

class TicketRequest(BaseModel):
    subject: str
    description: str

class RiskRequest(BaseModel):
    userId: str
    behavior: dict

class BurnoutRequest(BaseModel):
    userId: str
    metrics: dict

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        text = f"{request.subject} {request.description}"
        result = ticket_classifier.predict(text)
        
        # Auto-assign priority based on keywords
        priority = "high" if any(word in text.lower() for word in ['urgent', 'critical', 'down']) else "medium"
        
        return {
            "category": result['category'],
            "department": result['department'],
            "priority": priority,
            "confidence": result['confidence']
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskRequest):
    try:
        risk_score = risk_detector.calculate_risk(request.behavior)
        
        risk_level = "low"
        if risk_score > 0.7:
            risk_level = "high"
        elif risk_score > 0.4:
            risk_level = "medium"
        
        return {
            "risk_score": risk_score,
            "risk_level": risk_level,
            "factors": risk_detector.get_risk_factors(request.behavior),
            "userId": request.userId
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    try:
        score = burnout_analyzer.calculate_burnout(request.metrics)
        
        recommendation = "Workload is healthy"
        if score > 0.7:
            recommendation = "High burnout risk - immediate action needed"
        elif score > 0.5:
            recommendation = "Moderate burnout risk - reduce workload"
        
        return {
            "burnout_score": score,
            "level": "high" if score > 0.7 else "medium" if score > 0.5 else "low",
            "recommendation": recommendation,
            "userId": request.userId
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### Burnout Analyzer

```python
# ml-service/models/burnout_analyzer.py
class BurnoutAnalyzer:
    def calculate_burnout(self, metrics):
        """
        Calculate burnout score based on workload metrics
        """
        score = 0.0
        
        # Hours worked (weight: 0.4)
        hours = metrics.get('hours_worked_week', 40)
        if hours > 60:
            score += 0.4
        elif hours > 50:
            score += 0.25
        elif hours > 45:
            score += 0.15
        
        # Task completion rate (weight: 0.3)
        completed = metrics.get('tasks_completed', 0)
        overdue = metrics.get('tasks_overdue', 0)
        if overdue > completed:
            score += 0.3
        elif overdue > 0:
            score += 0.15
        
        # Weekend work (weight: 0.2)
        if metrics.get('weekend_work', False):
            score += 0.2
        
        # Meeting load (weight: 0.1)
        meetings = metrics.get('meetings_per_week', 0)
        if meetings > 20:
            score += 0.1
        elif meetings > 15:
            score += 0.05
        
        return min(score, 1.0)
```

## Configuration

### MongoDB Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection failed:', error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Environment Variables Template

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_jwt_secret_minimum_32_characters
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=development

# ml-service/.env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
CACHE_PREDICTIONS=true

# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// Check connection status
mongoose.connection.on('error', err => {
  console.error('MongoDB connection error:', err);
});

mongoose.connection.on('disconnected', () => {
  console.warn('MongoDB disconnected. Attempting to reconnect...');
});
```

### JWT Token Expiration

```javascript
// Implement token refresh
const refreshToken = async (oldToken) => {
  try {
    const decoded = jwt.verify(oldToken, process.env.JWT_SECRET, { ignoreExpiration: true });
    const user = await User.findById(decoded.id);
    
    if (!user) throw new Error('User not found');
    
    return jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRE });
  } catch (error) {
    throw new Error('Token refresh failed');
  }
};
```

### ML Service Not Responding

```python
# Add health check and logging
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request, call_next):
    logger.info(f"Request: {request.method} {request.url}")
    try:
        response = await call_next(request)
        return response
    except Exception as e:
        logger.error(f"Request failed: {str(e)}")
        raise
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization

```javascript
// Cache AI predictions
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 3600 });

const getCachedPrediction = async (key, fetchFn) => {
  const cached = cache.get(key);
  if (cached) return cached;
  
  const result = await fetchFn();
  cache.set(key, result);
  return result;
};

// Usage
const burnoutScore = await getCachedPrediction(
  `burnout_${userId}`,
  () => axios.post(`${ML_SERVICE_URL}/api/ml/burnout-analysis`, data)
);
```

This skill enables AI agents to help developers deploy and extend the Enterprise User Management System with AI Analytics, including authentication, task management, support tickets, and ML-powered insights for organizational efficiency.
