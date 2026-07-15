---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered insights for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with burnout detection"
  - "build user management with ticket classification"
  - "add AI risk prediction to user dashboard"
  - "configure ML service for user analytics"
  - "integrate AI assistant for enterprise management"
  - "deploy user management system with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack JavaScript application that provides enterprise-level user management with integrated AI/ML capabilities for smart insights, including risk detection, anomaly detection, burnout analysis, and predictive analytics.

## What This Project Does

The Enterprise User Management System combines traditional CRUD operations for user/task management with AI-powered analytics:

- **User Management**: Role-based access control, authentication via JWT
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-based classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics and audit logs

## Architecture

The system consists of three main components:

1. **Frontend**: React.js application (port 3000)
2. **Backend**: Node.js REST API with MongoDB (port 5000)
3. **ML Service**: FastAPI with scikit-learn and River (port 8000)

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install all components
# Backend
cd backend
npm install

# Frontend
cd ../frontend
npm install

# ML Service
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**

```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

**ML Service (.env)**

```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the System

### Start Backend

```bash
cd backend
npm start
# Development mode with hot reload
npm run dev
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Start Frontend

```bash
cd frontend
npm start
```

## Key API Endpoints

### Authentication

```javascript
// Login
POST /api/auth/login
Body: { email: string, password: string }
Response: { token: string, user: object }

// Register
POST /api/auth/register
Body: { name: string, email: string, password: string, role: string }
```

### User Management

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: 'Bearer <token>' }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
Body: { name: string, email: string, role: string }

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
Body: { 
  title: string, 
  description: string, 
  assignedTo: userId,
  status: 'todo' | 'in-progress' | 'done',
  priority: 'low' | 'medium' | 'high'
}

// Get user tasks
GET /api/tasks/user/:userId

// Update task status
PATCH /api/tasks/:id/status
Body: { status: string }

// Track time
POST /api/tasks/:id/time
Body: { duration: number }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Body: { 
  title: string, 
  description: string, 
  category: string,
  priority: string 
}

// AI Classification
POST /api/ml/classify-ticket
Body: { title: string, description: string }
Response: { category: string, priority: string, confidence: number }
```

### AI Analytics

```javascript
// Risk prediction
POST /api/ml/predict-risk
Body: { userId: string, metrics: object }
Response: { riskScore: number, factors: array }

// Burnout detection
GET /api/ml/burnout/:userId
Response: { burnoutScore: number, workloadMetrics: object }

// Anomaly detection
POST /api/ml/detect-anomaly
Body: { userId: string, activity: object }
Response: { isAnomaly: boolean, anomalyScore: number }

// Project insights
GET /api/ml/project-insights/:projectId
Response: { delayProbability: number, predictions: object }
```

## Frontend Usage Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Fetch user data
      axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`)
        .then(res => setUser(res.data))
        .catch(() => logout())
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  return { user, login, logout, loading };
};
```

### Task Board Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await axios.get(`/api/tasks/user/${userId}`);
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(`/api/tasks/${taskId}/status`, { status: newStatus });
    fetchTasks();
  };

  return (
    <div className="task-board">
      <Column title="To Do" tasks={tasks.todo} onMove={updateTaskStatus} />
      <Column title="In Progress" tasks={tasks.inProgress} onMove={updateTaskStatus} />
      <Column title="Done" tasks={tasks.done} onMove={updateTaskStatus} />
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    const [burnout, risk] = await Promise.all([
      axios.get(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout/${userId}`),
      axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, { userId })
    ]);
    
    setAnalytics({
      burnoutScore: burnout.data.burnoutScore,
      riskScore: risk.data.riskScore,
      riskFactors: risk.data.factors
    });
  };

  return (
    <div className="ai-analytics">
      <h3>AI Insights</h3>
      {analytics && (
        <>
          <div className="metric">
            <span>Burnout Risk:</span>
            <progress value={analytics.burnoutScore} max="100" />
          </div>
          <div className="metric">
            <span>Risk Score:</span>
            <progress value={analytics.riskScore} max="100" />
          </div>
          <ul>
            {analytics.riskFactors.map((factor, i) => (
              <li key={i}>{factor}</li>
            ))}
          </ul>
        </>
      )}
    </div>
  );
};
```

## Backend Patterns

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.createUser = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;
    const user = new User({ name, email, password, role });
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const user = await User.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    ).select('-password');
    res.json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

exports.authenticate = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'Authentication required' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Access denied' });
    }
    next();
  };
};
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: { type: String },
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
    ref: 'User' 
  },
  timeTracked: { type: Number, default: 0 }, // in minutes
  dueDate: { type: Date },
  tags: [String]
}, { timestamps: true });

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Load models
MODEL_PATH = os.getenv('MODEL_PATH', './models')

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    avgCompletionTime: float
    overdueCount: int
    workloadScore: float

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simulate ML classification (replace with actual model)
        text = f"{request.title} {request.description}".lower()
        
        categories = ['technical', 'billing', 'general', 'urgent']
        priorities = ['low', 'medium', 'high', 'critical']
        
        # Simple keyword-based classification
        if any(word in text for word in ['urgent', 'critical', 'emergency']):
            category = 'urgent'
            priority = 'critical'
        elif any(word in text for word in ['payment', 'invoice', 'billing']):
            category = 'billing'
            priority = 'medium'
        elif any(word in text for word in ['bug', 'error', 'crash', 'technical']):
            category = 'technical'
            priority = 'high'
        else:
            category = 'general'
            priority = 'low'
        
        return {
            "category": category,
            "priority": priority,
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Calculate risk score based on metrics
        risk_score = 0
        factors = []
        
        if request.overdueCount > 3:
            risk_score += 30
            factors.append("High number of overdue tasks")
        
        if request.avgCompletionTime > 48:
            risk_score += 25
            factors.append("Slow task completion rate")
        
        if request.workloadScore > 80:
            risk_score += 35
            factors.append("High workload detected")
        
        if request.taskCount > 20:
            risk_score += 10
            factors.append("Large task backlog")
        
        return {
            "riskScore": min(risk_score, 100),
            "factors": factors,
            "recommendation": "Consider task reallocation" if risk_score > 50 else "Normal operation"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/ml/burnout/{userId}")
async def detect_burnout(userId: str):
    try:
        # Simulate burnout detection
        # In production, fetch user metrics from database
        
        burnout_score = np.random.randint(0, 100)
        
        return {
            "burnoutScore": burnout_score,
            "workloadMetrics": {
                "tasksPerWeek": 25,
                "avgHoursPerDay": 9.5,
                "weekendWork": True
            },
            "status": "high" if burnout_score > 70 else "moderate" if burnout_score > 40 else "low"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(data: Dict):
    try:
        # Simple anomaly detection
        anomaly_score = np.random.random()
        is_anomaly = anomaly_score > 0.7
        
        return {
            "isAnomaly": is_anomaly,
            "anomalyScore": float(anomaly_score),
            "details": "Unusual activity pattern detected" if is_anomaly else "Normal activity"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Database Schema

```javascript
// User Schema
{
  name: String,
  email: String (unique),
  password: String (hashed),
  role: ['admin', 'user', 'manager'],
  department: String,
  createdAt: Date,
  updatedAt: Date
}

// Task Schema
{
  title: String,
  description: String,
  status: ['todo', 'in-progress', 'done'],
  priority: ['low', 'medium', 'high'],
  assignedTo: ObjectId (ref: User),
  createdBy: ObjectId (ref: User),
  timeTracked: Number,
  dueDate: Date,
  tags: [String]
}

// Ticket Schema
{
  title: String,
  description: String,
  category: String,
  priority: String,
  status: ['open', 'in-progress', 'resolved', 'closed'],
  createdBy: ObjectId (ref: User),
  assignedTo: ObjectId (ref: User),
  aiClassification: Object
}
```

## Common Workflows

### User Registration and Login

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await axios.post('/api/auth/register', {
    name: userData.name,
    email: userData.email,
    password: userData.password,
    role: 'user'
  });
  return response.data;
};

// Login
const loginUser = async (email, password) => {
  const response = await axios.post('/api/auth/login', { email, password });
  localStorage.setItem('token', response.data.token);
  axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
  return response.data.user;
};
```

### Task Creation with AI Priority

```javascript
const createTaskWithAI = async (taskData) => {
  // Create task
  const task = await axios.post('/api/tasks', taskData);
  
  // Get AI recommendation for priority
  const aiInsight = await axios.post('/api/ml/predict-task-priority', {
    title: taskData.title,
    description: taskData.description,
    assignedTo: taskData.assignedTo
  });
  
  // Update task with AI recommendation
  if (aiInsight.data.suggestedPriority !== taskData.priority) {
    await axios.patch(`/api/tasks/${task.data._id}`, {
      priority: aiInsight.data.suggestedPriority,
      aiRecommendation: aiInsight.data.reason
    });
  }
  
  return task.data;
};
```

### Ticket Classification Workflow

```javascript
const createTicketWithAI = async (ticketData) => {
  // AI classification
  const classification = await axios.post('/api/ml/classify-ticket', {
    title: ticketData.title,
    description: ticketData.description
  });
  
  // Create ticket with AI classification
  const ticket = await axios.post('/api/tickets', {
    ...ticketData,
    category: classification.data.category,
    priority: classification.data.priority,
    aiClassification: {
      confidence: classification.data.confidence,
      timestamp: new Date()
    }
  });
  
  return ticket.data;
};
```

## Troubleshooting

### Backend Connection Issues

```bash
# Check MongoDB connection
mongosh
use enterprise_user_mgmt
db.stats()

# Verify environment variables
node -e "console.log(process.env.MONGODB_URI)"
```

### JWT Authentication Errors

```javascript
// Verify token in backend
const jwt = require('jsonwebtoken');
const token = 'your_token_here';
try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  console.log('Token valid:', decoded);
} catch (error) {
  console.error('Token error:', error.message);
}
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/docs

# Test ML endpoint
curl -X POST http://localhost:8000/api/ml/classify-ticket \
  -H "Content-Type: application/json" \
  -d '{"title":"Bug report","description":"Application crashes"}'

# Check logs
tail -f ml-service/logs/app.log
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Database Performance

```javascript
// Add indexes for frequently queried fields
// backend/models/Task.js
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ createdAt: -1 });
taskSchema.index({ priority: 1, dueDate: 1 });
```

## Deployment

### Production Build

```bash
# Frontend
cd frontend
npm run build

# Backend (with PM2)
cd backend
npm install pm2 -g
pm2 start server.js --name "enterprise-backend"

# ML Service (with Gunicorn)
cd ml-service
pip install gunicorn
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
```

### Environment Variables for Production

```bash
# Backend
NODE_ENV=production
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=https://ml.yourdomain.com

# Frontend
REACT_APP_API_URL=https://api.yourdomain.com
REACT_APP_ML_API_URL=https://ml.yourdomain.com
```
