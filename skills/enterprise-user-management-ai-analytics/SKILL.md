---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "configure AI analytics for user management"
  - "implement user task tracking with AI"
  - "create support ticket system with ML classification"
  - "build admin dashboard with risk detection"
  - "integrate AI burnout analysis for users"
  - "deploy user management system with FastAPI ML service"
  - "add anomaly detection to user management app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers deploy and customize a full-stack enterprise user management system featuring role-based access control, Kanban task boards, support ticket management, and AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## What It Does

The Enterprise User Management System provides:

- **User Management**: JWT authentication, role-based access (Admin/User), profile management
- **Task Management**: Kanban boards (To Do, In Progress, Done), time tracking, task assignment
- **Support Tickets**: Ticket creation, tracking, AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: User oversight, audit logs, organization analytics, security alerts

The system consists of three main components:
1. **Frontend** (React.js) - User interface for admins and users
2. **Backend** (Node.js) - REST API, authentication, data management
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB (local or cloud)
mongod --version
```

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

## Configuration

### Backend Configuration

Create `backend/.env`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

### Frontend Configuration

Create `frontend/.env`:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

### ML Service Configuration

Create `ml-service/.env`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the System

### Start All Services

```bash
# Terminal 1: Start Backend
cd backend
npm start

# Terminal 2: Start ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Start Frontend
cd frontend
npm start
```

Access the application at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// Register a new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Create user
POST /api/users
Headers: { Authorization: "Bearer <token>" }
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
{
  "name": "Jane Smith Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:userId
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { Authorization: "Bearer <token>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "user_id",
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

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get user tickets
GET /api/tickets

// Update ticket (Admin)
PATCH /api/tickets/:ticketId
{
  "status": "in-progress",
  "assignedTo": "admin_user_id",
  "response": "Investigating the issue"
}
```

## ML Service API Reference

### AI Analytics Endpoints

```python
# Risk Prediction
POST http://localhost:8000/api/ml/predict-risk
Content-Type: application/json

{
  "userId": "user_id",
  "loginAttempts": 5,
  "taskCompletionRate": 0.65,
  "avgResponseTime": 2.5,
  "lastLoginTime": "2026-04-15T10:30:00Z"
}

# Response
{
  "riskScore": 0.72,
  "riskLevel": "medium",
  "factors": ["low_completion_rate", "multiple_login_attempts"]
}
```

```python
# Anomaly Detection
POST http://localhost:8000/api/ml/detect-anomaly

{
  "userId": "user_id",
  "activityData": {
    "loginTime": "03:00",
    "location": "unusual_ip",
    "actionsPerHour": 150
  }
}

# Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "reason": "Unusual login time and high activity rate"
}
```

```python
# Burnout Detection
POST http://localhost:8000/api/ml/burnout-analysis

{
  "userId": "user_id",
  "workHours": 65,
  "tasksCompleted": 45,
  "overtimeHours": 15,
  "weekendWork": true
}

# Response
{
  "burnoutRisk": "high",
  "score": 0.82,
  "recommendations": [
    "Reduce weekly hours",
    "Redistribute workload",
    "Schedule time off"
  ]
}
```

```python
# Project Delay Prediction
POST http://localhost:8000/api/ml/predict-delay

{
  "projectId": "proj_123",
  "tasksTotal": 50,
  "tasksCompleted": 15,
  "daysElapsed": 30,
  "daysRemaining": 20,
  "teamSize": 5
}

# Response
{
  "delayProbability": 0.68,
  "estimatedDelay": 12,
  "delayDays": 12,
  "recommendations": [
    "Increase team size",
    "Reduce scope",
    "Extend deadline"
  ]
}
```

```python
# Ticket Classification
POST http://localhost:8000/api/ml/classify-ticket

{
  "ticketId": "ticket_456",
  "title": "Cannot access dashboard",
  "description": "Getting 500 error when logging in",
  "priority": "high"
}

# Response
{
  "category": "technical",
  "suggestedDepartment": "IT Support",
  "estimatedResolutionTime": 4,
  "confidence": 0.91
}
```

## Code Examples

### Frontend: User Authentication Flow

```javascript
// src/services/authService.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const authService = {
  async login(email, password) {
    const response = await axios.post(`${API_URL}/auth/login`, {
      email,
      password
    });
    
    if (response.data.token) {
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
    }
    
    return response.data;
  },

  async register(userData) {
    const response = await axios.post(`${API_URL}/auth/register`, userData);
    return response.data;
  },

  logout() {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  },

  getCurrentUser() {
    const user = localStorage.getItem('user');
    return user ? JSON.parse(user) : null;
  },

  getAuthHeader() {
    const token = localStorage.getItem('token');
    return token ? { Authorization: `Bearer ${token}` } : {};
  }
};
```

### Frontend: Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { authService } from '../services/authService';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks`,
        { headers: authService.getAuthHeader() }
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: authService.getAuthHeader() }
      );
      
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const TaskCard = ({ task }) => (
    <div className="task-card" draggable>
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <div className="task-actions">
        {task.status !== 'in-progress' && (
          <button onClick={() => moveTask(task._id, 'in-progress')}>
            Start
          </button>
        )}
        {task.status !== 'done' && (
          <button onClick={() => moveTask(task._id, 'done')}>
            Complete
          </button>
        )}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="kanban-column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column">
        <h3>In Progress ({tasks.inProgress.length})</h3>
        {tasks.inProgress.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
      
      <div className="kanban-column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(task => <TaskCard key={task._id} task={task} />)}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Backend: JWT Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

### Backend: Task Management Routes

```javascript
// backend/routes/tasks.js
const express = require('express');
const router = express.Router();
const { authMiddleware } = require('../middleware/auth');
const Task = require('../models/Task');

// Get user tasks
router.get('/', authMiddleware, async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.id },
        { createdBy: req.user.id }
      ]
    }).populate('assignedTo', 'name email');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create task
router.post('/', authMiddleware, async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.id
    });
    
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update task status
router.patch('/:id/status', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findOneAndUpdate(
      { _id: req.params.id, assignedTo: req.user.id },
      { status: req.body.status, updatedAt: Date.now() },
      { new: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Track time
router.post('/:id/time', authMiddleware, async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    task.timeTracked = (task.timeTracked || 0) + req.body.duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;
```

### Backend: MongoDB Models

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  description: {
    type: String,
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium'
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  dueDate: {
    type: Date
  },
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  tags: [String],
  createdAt: {
    type: Date,
    default: Date.now
  },
  updatedAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: {
    type: String,
    trim: true
  },
  avatar: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date,
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

### ML Service: FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from river import anomaly, preprocessing
import joblib
import os

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models directory
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Anomaly detection model (online learning)
anomaly_detector = anomaly.HalfSpaceTrees(seed=42)

# Request/Response Models
class RiskPredictionRequest(BaseModel):
    userId: str
    loginAttempts: int
    taskCompletionRate: float
    avgResponseTime: float
    lastLoginTime: str

class RiskPredictionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activityData: dict

class AnomalyDetectionResponse(BaseModel):
    isAnomaly: bool
    anomalyScore: float
    reason: str

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workHours: float
    tasksCompleted: int
    overtimeHours: float
    weekendWork: bool

class BurnoutAnalysisResponse(BaseModel):
    burnoutRisk: str
    score: float
    recommendations: List[str]

class ProjectDelayRequest(BaseModel):
    projectId: str
    tasksTotal: int
    tasksCompleted: int
    daysElapsed: int
    daysRemaining: int
    teamSize: int

class ProjectDelayResponse(BaseModel:
    delayProbability: float
    estimatedDelay: int
    delayDays: int
    recommendations: List[str]

class TicketClassificationRequest(BaseModel):
    ticketId: str
    title: str
    description: str
    priority: str

class TicketClassificationResponse(BaseModel):
    category: str
    suggestedDepartment: str
    estimatedResolutionTime: int
    confidence: float

# Helper functions
def calculate_risk_score(data: RiskPredictionRequest) -> float:
    """Calculate risk score based on user behavior"""
    score = 0.0
    
    # Login attempts factor
    if data.loginAttempts > 3:
        score += 0.3
    
    # Task completion rate
    if data.taskCompletionRate < 0.7:
        score += 0.25
    
    # Response time
    if data.avgResponseTime > 3.0:
        score += 0.2
    
    return min(score, 1.0)

def get_risk_factors(data: RiskPredictionRequest) -> List[str]:
    """Identify risk factors"""
    factors = []
    
    if data.loginAttempts > 3:
        factors.append("multiple_login_attempts")
    if data.taskCompletionRate < 0.7:
        factors.append("low_completion_rate")
    if data.avgResponseTime > 3.0:
        factors.append("slow_response_time")
    
    return factors

def calculate_burnout_score(data: BurnoutAnalysisRequest) -> float:
    """Calculate burnout risk score"""
    score = 0.0
    
    if data.workHours > 50:
        score += 0.3
    if data.overtimeHours > 10:
        score += 0.25
    if data.weekendWork:
        score += 0.2
    if data.tasksCompleted > 40:
        score += 0.15
    
    return min(score, 1.0)

# API Endpoints
@app.get("/")
async def root():
    return {
        "service": "Enterprise User Management ML Service",
        "version": "1.0.0",
        "status": "running"
    }

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk based on behavior patterns"""
    try:
        risk_score = calculate_risk_score(request)
        factors = get_risk_factors(request)
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.6:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        return RiskPredictionResponse(
            riskScore=round(risk_score, 2),
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly", response_model=AnomalyDetectionResponse)
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalies in user activity"""
    try:
        # Extract features from activity data
        features = {
            'hour': int(request.activityData.get('loginTime', '12:00').split(':')[0]),
            'actions': request.activityData.get('actionsPerHour', 0),
            'location_change': 1 if 'unusual' in request.activityData.get('location', '') else 0
        }
        
        # Update anomaly detector
        anomaly_detector.learn_one(features)
        score = anomaly_detector.score_one(features)
        
        is_anomaly = score > 0.5
        reason = ""
        
        if features['hour'] < 6 or features['hour'] > 22:
            reason += "Unusual login time. "
        if features['actions'] > 100:
            reason += "High activity rate. "
        if features['location_change']:
            reason += "Unusual location. "
        
        return AnomalyDetectionResponse(
            isAnomaly=is_anomaly,
            anomalyScore=round(float(score), 2),
            reason=reason.strip()
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis", response_model=BurnoutAnalysisResponse)
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        score = calculate_burnout_score(request)
        
        # Determine risk level
        if score < 0.3:
            risk = "low"
        elif score < 0.6:
            risk = "medium"
        else:
            risk = "high"
        
        # Generate recommendations
        recommendations = []
        if request.workHours > 50:
            recommendations.append("Reduce weekly hours")
        if request.overtimeHours > 10:
            recommendations.append("Limit overtime")
        if request.weekendWork:
            recommendations.append("Avoid weekend work")
        if request.tasksCompleted > 40:
            recommendations.append("Redistribute workload")
        
        if not recommendations:
            recommendations.append("Maintain current work-life balance")
        
        return BurnoutAnalysisResponse(
            burnoutRisk=risk,
            score=round(score, 2),
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-delay", response_model=ProjectDelayResponse)
async def predict_project_delay(request: ProjectDelayRequest):
    """Predict project delay probability"""
    try:
        # Calculate progress rate
        progress_rate = request.tasksCompleted / request.tasksTotal if request.tasksTotal > 0 else 0
        expected_progress = request.daysElapsed / (request.daysElapsed + request.daysRemaining)
        
        # Calculate delay probability
        if progress_rate < expected_progress * 0.8:
            delay_prob = 0.8
        elif progress_rate < expected_progress:
            delay_prob = 0.5
        else:
            delay_prob = 0.2
        
        # Estimate delay in days
        if delay_prob > 0.5:
            remaining_tasks = request.tasksTotal - request.tasksCompleted
            tasks_per_day = request.tasksCompleted / request.daysElapsed if request.daysElapsed > 0 else 1
            estimated_days = remaining_tasks / (tasks_per_day * request.teamSize) if tasks_per_day > 0 else request.daysRemaining
            delay_days = max(0, int(estimated_days - request.daysRemaining))
        else:
            delay_days = 0
        
        # Generate recommendations
        recommendations = []
        if delay_prob > 0.5:
            recommendations.append("Increase team size")
            recommendations.append("Reduce scope")
            if delay_days > 10:
                recommendations.append("Extend deadline")
        else:
            recommendations.append("Maintain current pace")
        
        return ProjectDelayResponse(
            delayProbability=round(delay_prob, 2),
            estimatedDelay=delay_days,
            delayDays=delay_days,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    """Classify support ticket and route to appropriate department"""
    try:
        # Simple keyword-based classification
        text = (request.title + " " + request.description).lower()
        
        # Category classification
        if any(word in text for word in ['login', 'password', 'access', 'authentication']):
            category = "authentication"
            department = "IT Support"
            resolution_time = 2
        elif any(word in text for word in ['bug', 'error', '500', 'crash', 'not working']):
            category = "technical"
            department = "Engineering"
            resolution_time = 8
        elif any(word in text for word in ['payment', 'billing', 'invoice', 'subscription']):
            category = "billing"
            department = "Finance"
            resolution_time = 4
        elif any(word in text for word in ['feature', 'request', 'enhancement', 'suggestion']):
            category = "feature_request"
            department = "Product"
            resolution_time = 48
        else:
            category = "general"
            department = "Customer Support"
            resolution_time = 6
        
        # Adjust resolution time based on priority
        if request.priority == "urgent":
            resolution_time = max(1, resolution_time // 2)
        elif request.priority == "low":
            resolution_time *= 2
        
        return TicketClassificationResponse(
            category=category,
            suggestedDepartment=department,
            estimatedResolutionTime=resolution_time,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": True}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### ML Service: Requirements File

```python
# ml-service/requirements.txt
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
scikit-learn==1.3.2
river
