---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with AI insights"
  - "build task management system with burnout detection"
  - "integrate AI ticket classification system"
  - "deploy enterprise user management with FastAPI ML service"
  - "configure user management with risk prediction"
  - "develop admin dashboard with AI analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application that combines traditional user/task management with AI-powered analytics. It provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Architecture:**
- **Frontend**: React.js with JWT authentication
- **Backend**: Node.js/Express REST API
- **ML Service**: FastAPI with scikit-learn and River (online learning)
- **Database**: MongoDB

## Installation

### Prerequisites

```bash
# Ensure you have installed:
node --version  # v14+ required
python --version  # 3.8+ required
mongo --version  # MongoDB 4.4+ required
```

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOL
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOL

npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOL
MODEL_PATH=./models
LOG_LEVEL=info
EOL

uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOL
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOL

npm start
```

## Key API Endpoints

### Authentication APIs

```javascript
// Backend: POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}
// Response: { "token": "jwt_token", "user": {...} }

// Backend: POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass",
  "role": "user"
}
```

### User Management APIs

```javascript
// GET /api/users - List all users (Admin only)
// GET /api/users/:id - Get user details
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (Admin only)

// Example: Fetch users with authentication
const fetchUsers = async () => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${localStorage.getItem('token')}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};
```

### Task Management APIs

```javascript
// POST /api/tasks - Create task
{
  "title": "Implement feature X",
  "description": "Details...",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
{
  "status": "in-progress",
  "timeSpent": 120  // minutes
}

// GET /api/tasks/user/:userId - Get user's tasks
```

### Support Ticket APIs

```javascript
// POST /api/tickets - Create support ticket
{
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "high",
  "category": "technical"
}

// GET /api/tickets - List all tickets
// PUT /api/tickets/:id - Update ticket status
```

### AI/ML Service APIs

```python
# FastAPI ML endpoints

# POST /api/ml/classify-ticket - AI ticket classification
{
  "title": "Cannot login to system",
  "description": "Getting authentication error",
  "category": "technical"
}
# Response: { "predicted_priority": "high", "predicted_category": "authentication" }

# POST /api/ml/risk-detection - User risk assessment
{
  "user_id": "123",
  "activity_data": {
    "login_attempts": 5,
    "failed_logins": 3,
    "tasks_completed": 2,
    "avg_completion_time": 480
  }
}
# Response: { "risk_score": 0.75, "risk_level": "high", "factors": [...] }

# POST /api/ml/burnout-detection - Analyze user burnout
{
  "user_id": "123",
  "workload_data": {
    "tasks_assigned": 15,
    "tasks_completed": 5,
    "avg_daily_hours": 10.5,
    "weekend_work": true
  }
}
# Response: { "burnout_score": 0.82, "risk_level": "critical" }

# POST /api/ml/anomaly-detection - Detect unusual behavior
{
  "user_id": "123",
  "behavior_data": {
    "login_time": "03:00",
    "location": "unusual_location",
    "access_pattern": "abnormal"
  }
}
# Response: { "is_anomaly": true, "confidence": 0.89 }
```

## Frontend React Components

### Protected Route with Authentication

```javascript
// src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;
```

### User Dashboard Component

```javascript
// src/components/UserDashboard.js
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [aiInsights, setAiInsights] = useState(null);
  
  useEffect(() => {
    fetchUserTasks();
    fetchAIInsights();
  }, []);
  
  const fetchUserTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const user = JSON.parse(localStorage.getItem('user'));
      
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${user.id}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      setTasks(response.data);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };
  
  const fetchAIInsights = async () => {
    try {
      const token = localStorage.getItem('token');
      const user = JSON.parse(localStorage.getItem('user'));
      
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
        {
          user_id: user.id,
          workload_data: {
            tasks_assigned: tasks.length,
            tasks_completed: tasks.filter(t => t.status === 'done').length,
            avg_daily_hours: 8,
            weekend_work: false
          }
        },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      setAiInsights(response.data);
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    }
  };
  
  return (
    <div className="dashboard">
      <h1>User Dashboard</h1>
      
      {aiInsights && (
        <div className="ai-insights">
          <h3>Burnout Risk: {aiInsights.risk_level}</h3>
          <p>Score: {aiInsights.burnout_score}</p>
        </div>
      )}
      
      <div className="kanban-board">
        <div className="column">
          <h3>To Do</h3>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <div key={task.id} className="task-card">{task.title}</div>
          ))}
        </div>
        <div className="column">
          <h3>In Progress</h3>
          {tasks.filter(t => t.status === 'in-progress').map(task => (
            <div key={task.id} className="task-card">{task.title}</div>
          ))}
        </div>
        <div className="column">
          <h3>Done</h3>
          {tasks.filter(t => t.status === 'done').map(task => (
            <div key={task.id} className="task-card">{task.title}</div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

## Backend Node.js Implementation

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const registerUser = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }
    
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user'
    });
    
    await user.save();
    
    const token = jwt.sign(
      { id: user._id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.status(201).json({ token, user: { id: user._id, name, email, role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

const loginUser = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(401).json({ message: 'Invalid credentials' });
    }
    
    const token = jwt.sign(
      { id: user._id, email: user.email, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );
    
    res.json({ token, user: { id: user._id, name: user.name, email, role: user.role } });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

module.exports = { registerUser, loginUser };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, status, priority, dueDate } = req.body;
    
    const task = new Task({
      title,
      description,
      assignedTo,
      status: status || 'todo',
      priority,
      dueDate,
      createdBy: req.user.id
    });
    
    await task.save();
    
    // Get AI prediction for task complexity
    try {
      const mlResponse = await axios.post(
        `${process.env.ML_SERVICE_URL}/api/ml/predict-complexity`,
        { title, description }
      );
      task.predictedComplexity = mlResponse.data.complexity;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError.message);
    }
    
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error creating task', error: error.message });
  }
};

const updateTask = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const task = await Task.findByIdAndUpdate(id, updates, { new: true });
    
    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Error updating task', error: error.message });
  }
};

module.exports = { createTask, updateTask };
```

## ML Service FastAPI Implementation

### Main FastAPI Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
import joblib
import numpy as np
from river import anomaly, tree
import os

app = FastAPI(title="Enterprise ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models storage
MODEL_PATH = os.getenv('MODEL_PATH', './models')
os.makedirs(MODEL_PATH, exist_ok=True)

# Initialize online learning model for anomaly detection
anomaly_detector = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    category: Optional[str] = None

class RiskDetectionRequest(BaseModel):
    user_id: str
    activity_data: dict

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    workload_data: dict

class AnomalyDetectionRequest(BaseModel):
    user_id: str
    behavior_data: dict

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Classify support ticket priority and category using ML
    """
    try:
        # Simple rule-based classifier (replace with trained model)
        text = f"{request.title} {request.description}".lower()
        
        # Priority classification
        high_priority_keywords = ['urgent', 'critical', 'down', 'error', 'crash', 'bug']
        medium_priority_keywords = ['issue', 'problem', 'help', 'question']
        
        priority = 'low'
        if any(keyword in text for keyword in high_priority_keywords):
            priority = 'high'
        elif any(keyword in text for keyword in medium_priority_keywords):
            priority = 'medium'
        
        # Category classification
        categories = {
            'authentication': ['login', 'password', 'access', 'credentials'],
            'technical': ['error', 'bug', 'crash', 'performance'],
            'feature': ['request', 'feature', 'enhancement'],
            'general': []
        }
        
        predicted_category = 'general'
        for category, keywords in categories.items():
            if any(keyword in text for keyword in keywords):
                predicted_category = category
                break
        
        return {
            "predicted_priority": priority,
            "predicted_category": predicted_category,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    """
    Detect user risk based on activity patterns
    """
    try:
        activity = request.activity_data
        
        # Calculate risk score based on multiple factors
        risk_factors = []
        risk_score = 0.0
        
        # Failed login attempts
        failed_logins = activity.get('failed_logins', 0)
        if failed_logins > 3:
            risk_score += 0.3
            risk_factors.append('Multiple failed login attempts')
        
        # Task completion rate
        tasks_completed = activity.get('tasks_completed', 0)
        tasks_assigned = activity.get('tasks_assigned', 1)
        completion_rate = tasks_completed / tasks_assigned
        
        if completion_rate < 0.3:
            risk_score += 0.2
            risk_factors.append('Low task completion rate')
        
        # Average completion time
        avg_time = activity.get('avg_completion_time', 240)
        if avg_time > 600:  # More than 10 hours
            risk_score += 0.2
            risk_factors.append('Unusually long task completion times')
        
        # Determine risk level
        if risk_score >= 0.7:
            risk_level = 'high'
        elif risk_score >= 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            "risk_score": min(risk_score, 1.0),
            "risk_level": risk_level,
            "factors": risk_factors,
            "user_id": request.user_id
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    """
    Analyze user workload and detect burnout risk
    """
    try:
        workload = request.workload_data
        
        burnout_score = 0.0
        factors = []
        
        # Tasks assigned vs completed
        tasks_assigned = workload.get('tasks_assigned', 0)
        tasks_completed = workload.get('tasks_completed', 0)
        
        if tasks_assigned > 10 and (tasks_completed / tasks_assigned) < 0.5:
            burnout_score += 0.3
            factors.append('High task backlog')
        
        # Average daily hours
        avg_hours = workload.get('avg_daily_hours', 8)
        if avg_hours > 10:
            burnout_score += 0.3
            factors.append('Excessive working hours')
        
        # Weekend work
        if workload.get('weekend_work', False):
            burnout_score += 0.2
            factors.append('Regular weekend work')
        
        # Overtime frequency
        overtime_days = workload.get('overtime_days', 0)
        if overtime_days > 3:
            burnout_score += 0.2
            factors.append('Frequent overtime')
        
        # Determine risk level
        if burnout_score >= 0.7:
            risk_level = 'critical'
        elif burnout_score >= 0.5:
            risk_level = 'high'
        elif burnout_score >= 0.3:
            risk_level = 'moderate'
        else:
            risk_level = 'low'
        
        return {
            "burnout_score": min(burnout_score, 1.0),
            "risk_level": risk_level,
            "factors": factors,
            "user_id": request.user_id,
            "recommendation": "Consider workload redistribution" if burnout_score > 0.5 else "Workload appears manageable"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """
    Detect anomalous user behavior using online learning
    """
    try:
        behavior = request.behavior_data
        
        # Convert behavior data to feature vector
        features = {
            'login_hour': int(behavior.get('login_time', '09:00').split(':')[0]),
            'location_hash': hash(behavior.get('location', 'normal')) % 100,
            'access_pattern_score': 1 if behavior.get('access_pattern') == 'abnormal' else 0
        }
        
        # Score with anomaly detector
        score = anomaly_detector.score_one(features)
        
        # Update model with new observation
        anomaly_detector.learn_one(features)
        
        # Threshold for anomaly
        is_anomaly = score > 0.6
        
        return {
            "is_anomaly": is_anomaly,
            "confidence": float(score),
            "user_id": request.user_id,
            "alert_level": "high" if score > 0.8 else "medium" if score > 0.6 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "service": "ML Service"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Database Models

### MongoDB User Schema

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  department: String,
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date
});

module.exports = mongoose.model('User', userSchema);
```

### MongoDB Task Schema

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  status: {
    type: String,
    enum: ['todo', 'in-progress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
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
  dueDate: Date,
  timeSpent: {
    type: Number,
    default: 0  // in minutes
  },
  predictedComplexity: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### MongoDB Ticket Schema

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  category: {
    type: String,
    enum: ['technical', 'authentication', 'feature', 'general'],
    default: 'general'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  aiPredictedPriority: String,
  aiPredictedCategory: String,
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## Common Patterns

### Integrating AI Predictions in Workflow

```javascript
// backend/services/aiService.js
const axios = require('axios');

class AIService {
  constructor() {
    this.mlServiceUrl = process.env.ML_SERVICE_URL || 'http://localhost:8000';
  }
  
  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/api/ml/classify-ticket`,
        ticketData
      );
      return response.data;
    } catch (error) {
      console.error('AI classification failed:', error.message);
      return null;
    }
  }
  
  async detectUserRisk(userId, activityData) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/api/ml/risk-detection`,
        { user_id: userId, activity_data: activityData }
      );
      return response.data;
    } catch (error) {
      console.error('Risk detection failed:', error.message);
      return null;
    }
  }
  
  async checkBurnout(userId, workloadData) {
    try {
      const response = await axios.post(
        `${this.mlServiceUrl}/api/ml/burnout-detection`,
        { user_id: userId, workload_data: workloadData }
      );
      return response.data;
    } catch (error) {
      console.error('Burnout detection failed:', error.message);
      return null;
    }
  }
}

module.exports = new AIService();
```

### Using AI Service in Controllers

```javascript
// backend/controllers/ticketController.js
const Ticket = require('../models/Ticket');
const aiService = require('../services/aiService');

const createTicket = async (req, res) => {
  try {
    const { title, description, category } = req.body;
    
    // Get AI prediction
    const aiPrediction = await aiService.classifyTicket({
      title,
      description,
      category
    });
    
    const ticket = new Ticket({
      title,
      description,
      category: aiPrediction?.predicted_category || category || 'general',
      priority: aiPrediction?.predicted_priority || 'medium',
      aiPredictedPriority: aiPrediction?.predicted_priority,
      aiPredictedCategory: aiPrediction?.predicted_category,
      createdBy: req.user.id
    });
    
    await ticket.save();
    
    res.status(201).json(ticket);
  } catch (error) {
    res.status(500).json({ message: 'Error creating ticket', error: error.message });
  }
};

module.exports = { createTicket };
```

## Configuration

### Environment Variables

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
LOG_LEVEL=info
```

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
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Troubleshooting

### Common Issues

**1. JWT Token Expiration**
```javascript
// Handle token refresh
const refreshToken = async () => {
  try {
    const response = await axios.post('/api/auth/refresh', {
      refreshToken: localStorage.getItem('refreshToken')
    });
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

**2. ML Service Connection Issues**
```python
# Add health check and retry logic
from fastapi import HTTPException
import http
