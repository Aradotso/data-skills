---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with AI insights"
  - "create user dashboard with task tracking"
  - "build admin panel with user management"
  - "add AI-powered ticket classification"
  - "implement burnout detection and risk prediction"
  - "configure kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. It provides user and admin dashboards, task management with Kanban boards, support ticket systems, and ML-based insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## Architecture Overview

The system consists of three main components:

1. **Frontend** (React.js) - User and admin interfaces
2. **Backend** (Node.js/Express) - REST API and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics engine

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
python --version  # Python 3.8+ required
mongodb --version  # MongoDB 4.4+ required
```

### Full Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
cp .env.example .env
# Edit .env with your configuration

# ML Service setup
cd ../ml-service
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install
```

## Configuration

### Backend Environment Variables (.env)

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT Authentication
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${EMAIL_USER}
SMTP_PASS=${EMAIL_PASSWORD}
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    app_name: str = "Enterprise AI Analytics"
    mongodb_uri: str = os.getenv("MONGODB_URI", "mongodb://localhost:27017/enterprise_user_mgmt")
    model_path: str = "./models"
    
    # AI Configuration
    risk_threshold: float = 0.7
    anomaly_threshold: float = 2.5
    burnout_threshold: float = 0.65
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### Frontend Configuration (frontend/src/config.js)

```javascript
export const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';
export const ML_BASE_URL = process.env.REACT_APP_ML_URL || 'http://localhost:8000';

export const ROLES = {
  ADMIN: 'admin',
  USER: 'user',
  MANAGER: 'manager'
};

export const TASK_STATUS = {
  TODO: 'todo',
  IN_PROGRESS: 'in_progress',
  DONE: 'done'
};
```

## Running the System

### Start Backend Server

```bash
cd backend
npm start
# Development mode with auto-reload
npm run dev
```

### Start ML Service

```bash
cd ml-service
source venv/bin/activate
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Start Frontend

```bash
cd frontend
npm start
# Runs on http://localhost:3000
```

## API Reference

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Authorization: Bearer ${JWT_TOKEN}

// Create user
POST /api/users
Authorization: Bearer ${JWT_TOKEN}
{
  "username": "jane.smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Authorization: Bearer ${JWT_TOKEN}
{
  "role": "manager",
  "isActive": true
}

// Delete user
DELETE /api/users/:userId
Authorization: Bearer ${JWT_TOKEN}
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks?userId=123&status=in_progress
Authorization: Bearer ${JWT_TOKEN}

// Create task
POST /api/tasks
Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Implement feature X",
  "description": "Add new authentication flow",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "estimatedHours": 8
}

// Update task status
PATCH /api/tasks/:taskId/status
{
  "status": "in_progress",
  "timeSpent": 120  // minutes
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer ${JWT_TOKEN}
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error when accessing admin panel",
  "priority": "high",
  "category": "technical"
}

// Get tickets (auto-classified by AI)
GET /api/tickets?status=open&assignedTo=auto
```

### AI Analytics Endpoints

```python
# Risk Prediction
POST http://localhost:8000/api/ai/predict-risk
Content-Type: application/json

{
  "userId": "user_123",
  "loginAttempts": 5,
  "failedLogins": 2,
  "tasksCompleted": 45,
  "averageTaskTime": 180,
  "ticketsRaised": 8
}
# Returns: { "riskScore": 0.34, "riskLevel": "low", "factors": [...] }

# Burnout Detection
POST http://localhost:8000/api/ai/detect-burnout
{
  "userId": "user_123",
  "weeklyHours": 52,
  "tasksInProgress": 12,
  "overdueTasksCount": 3,
  "ticketsRaised": 5,
  "timeToComplete": 8.5
}
# Returns: { "burnoutScore": 0.72, "risk": "high", "recommendations": [...] }

# Anomaly Detection
POST http://localhost:8000/api/ai/detect-anomaly
{
  "userId": "user_123",
  "loginTime": "2026-04-15T03:30:00Z",
  "ipAddress": "192.168.1.100",
  "location": "Unknown",
  "deviceType": "mobile"
}
# Returns: { "isAnomaly": true, "score": 3.2, "reason": "Unusual login time" }

# Ticket Classification
POST http://localhost:8000/api/ai/classify-ticket
{
  "subject": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "high"
}
# Returns: { "category": "technical", "department": "IT", "estimatedResolutionTime": 2.5 }
```

## Frontend Implementation Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect, useContext, createContext } from 'react';
import { API_BASE_URL } from '../config';

const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      fetchUser(token);
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async (token) => {
    try {
      const response = await fetch(`${API_BASE_URL}/auth/me`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${API_BASE_URL}/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { API_BASE_URL } from '../config';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${API_BASE_URL}/tasks?userId=${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      in_progress: data.filter(t => t.status === 'in_progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${API_BASE_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in_progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard Component

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { ML_BASE_URL } from '../config';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutScore: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);

  const fetchAIAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch risk prediction
    const riskResponse = await fetch(`${ML_BASE_URL}/api/ai/predict-risk`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ userId })
    });
    const riskData = await riskResponse.json();

    // Fetch burnout detection
    const burnoutResponse = await fetch(`${ML_BASE_URL}/api/ai/detect-burnout`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ userId })
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskScore: riskData.riskScore,
      burnoutScore: burnoutData.burnoutScore,
      anomalies: riskData.factors || []
    });
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      <div className="metric-card">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.riskScore > 0.7 ? 'high' : 'low'}`}>
          {(analytics.riskScore * 100).toFixed(1)}%
        </div>
      </div>

      <div className="metric-card">
        <h3>Burnout Risk</h3>
        <div className={`score ${analytics.burnoutScore > 0.65 ? 'high' : 'low'}`}>
          {(analytics.burnoutScore * 100).toFixed(1)}%
        </div>
      </div>

      {analytics.anomalies.length > 0 && (
        <div className="anomalies">
          <h3>Detected Issues</h3>
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>{anomaly}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend Implementation Patterns

### User Model (MongoDB Schema)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['admin', 'user', 'manager'],
    default: 'user'
  },
  department: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
  loginAttempts: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ message: 'Not authorized, no token' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    res.status(401).json({ message: 'Not authorized, token failed' });
  }
};

const admin = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, admin };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.getTasks = async (req, res) => {
  try {
    const { userId, status, priority } = req.query;
    const filter = {};
    
    if (userId) filter.assignedTo = userId;
    if (status) filter.status = status;
    if (priority) filter.priority = priority;

    const tasks = await Task.find(filter)
      .populate('assignedTo', 'username email')
      .populate('createdBy', 'username')
      .sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { taskId } = req.params;
    const { status, timeSpent } = req.body;

    const task = await Task.findById(taskId);
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    task.status = status;
    if (timeSpent) {
      task.timeSpent = (task.timeSpent || 0) + timeSpent;
    }
    
    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
        
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            return joblib.load(self.model_path)
        else:
            # Initialize new model
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            return model
    
    def predict(self, features):
        """
        Features: [loginAttempts, failedLogins, tasksCompleted, 
                   averageTaskTime, ticketsRaised, workHours]
        """
        feature_array = np.array(features).reshape(1, -1)
        
        # Get probability scores
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(feature_array)[0][1]
        else:
            # Fallback to simple heuristic
            risk_score = self._calculate_heuristic_risk(features)
        
        risk_level = self._classify_risk(risk_score)
        factors = self._identify_risk_factors(features)
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def _calculate_heuristic_risk(self, features):
        """Calculate risk using weighted heuristics"""
        weights = [0.3, 0.4, -0.1, 0.1, 0.15, 0.05]
        normalized = [
            min(features[0] / 10, 1),  # loginAttempts
            min(features[1] / 5, 1),   # failedLogins
            1 - min(features[2] / 100, 1),  # tasksCompleted (inverse)
            min(features[3] / 480, 1),  # averageTaskTime
            min(features[4] / 20, 1),   # ticketsRaised
            min(features[5] / 60, 1)    # workHours
        ]
        return sum(w * n for w, n in zip(weights, normalized))
    
    def _classify_risk(self, score):
        if score >= 0.7:
            return 'high'
        elif score >= 0.4:
            return 'medium'
        else:
            return 'low'
    
    def _identify_risk_factors(self, features):
        factors = []
        if features[1] > 3:
            factors.append('Multiple failed login attempts')
        if features[0] > 8:
            factors.append('Unusual number of login sessions')
        if features[4] > 15:
            factors.append('High number of support tickets')
        if features[5] > 50:
            factors.append('Excessive working hours')
        return factors
    
    def save_model(self):
        os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
        joblib.dump(self.model, self.model_path)
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np

class BurnoutDetector:
    def __init__(self, threshold=0.65):
        self.threshold = threshold
    
    def detect(self, user_data):
        """
        user_data: {
            'weeklyHours': float,
            'tasksInProgress': int,
            'overdueTasksCount': int,
            'ticketsRaised': int,
            'timeToComplete': float
        }
        """
        score = self._calculate_burnout_score(user_data)
        risk = self._classify_risk(score)
        recommendations = self._generate_recommendations(user_data, score)
        
        return {
            'burnoutScore': float(score),
            'risk': risk,
            'recommendations': recommendations
        }
    
    def _calculate_burnout_score(self, data):
        # Normalized metrics
        hours_factor = min(data.get('weeklyHours', 40) / 60, 1.0) * 0.35
        tasks_factor = min(data.get('tasksInProgress', 0) / 15, 1.0) * 0.25
        overdue_factor = min(data.get('overdueTasksCount', 0) / 5, 1.0) * 0.20
        tickets_factor = min(data.get('ticketsRaised', 0) / 10, 1.0) * 0.10
        completion_factor = min(data.get('timeToComplete', 5) / 10, 1.0) * 0.10
        
        return hours_factor + tasks_factor + overdue_factor + tickets_factor + completion_factor
    
    def _classify_risk(self, score):
        if score >= 0.75:
            return 'critical'
        elif score >= self.threshold:
            return 'high'
        elif score >= 0.4:
            return 'medium'
        else:
            return 'low'
    
    def _generate_recommendations(self, data, score):
        recommendations = []
        
        if data.get('weeklyHours', 0) > 45:
            recommendations.append('Reduce weekly working hours to below 45')
        
        if data.get('tasksInProgress', 0) > 10:
            recommendations.append('Delegate or postpone some tasks')
        
        if data.get('overdueTasksCount', 0) > 3:
            recommendations.append('Review and reschedule overdue tasks')
        
        if score >= self.threshold:
            recommendations.append('Schedule time off or reduce workload')
            recommendations.append('Consult with manager about workload distribution')
        
        return recommendations
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier

app = FastAPI(title="Enterprise AI Analytics Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int = 0
    failedLogins: int = 0
    tasksCompleted: int = 0
    averageTaskTime: float = 0
    ticketsRaised: int = 0
    workHours: float = 40

class BurnoutDetectionRequest(BaseModel):
    userId: str
    weeklyHours: float
    tasksInProgress: int
    overdueTasksCount: int
    ticketsRaised: int
    timeToComplete: float

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str
    priority: str

# Endpoints
@app.post("/api/ai/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = [
            request.loginAttempts,
            request.failedLogins,
            request.tasksCompleted,
            request.averageTaskTime,
            request.ticketsRaised,
            request.workHours
        ]
        result = risk_predictor.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        user_data = request.dict()
        result = burnout_detector.detect(user_data)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ai/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        ticket_data = request.dict()
        result = ticket_classifier.classify(ticket_data)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/ai/health")
async def health_check():
    return {
        "status": "healthy",
        "models": {
            "risk_predictor": "loaded",
            "burnout_detector": "loaded",
            "ticket_classifier": "loaded"
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Workflows

### User Registration and Login Flow

```javascript
// Complete user authentication flow
async function registerAndLogin() {
  const API_URL = 'http://localhost:5000/api';
  
  // 1. Register new user
  const registerResponse = await fetch(`${API_URL}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: 'john.doe',
      email: 'john@company.com',
      password: 'SecurePass123!',
      role: 'user',
      department: 'Engineering'
    })
  });
  
  if (!registerResponse.ok) {
    throw new Error('Registration failed');
  }
  
  // 2. Login
  const loginResponse = await fetch(`${API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: 'john@company.com',
      password: 'SecurePass123!'
    })
  });
  
  const { token, user } = await loginResponse.json();
  
  // 3. Store token
  localStorage.setItem('token', token);
  
  // 4. Fetch user profile
  const profileResponse = await fetch(`${API_URL}/users/${user._id}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  
  const profile = await profileResponse.json();
  return profile;
}
```

### Complete Task Management Workflow

```javascript
// Task lifecycle management
class TaskManager {
  constructor(apiUrl, token) {
    this.apiUrl = apiUrl;
    this.token = token;
  }
  
  async createTask(taskData) {
    const response = await fetch(`${this.apiUrl}/tasks`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(taskData)
    });
    return response.json();
  }
  
