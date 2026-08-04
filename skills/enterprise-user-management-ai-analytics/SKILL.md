---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user dashboard with task tracking"
  - "implement AI ticket classification system"
  - "add burnout detection to user management"
  - "build admin dashboard with ML insights"
  - "configure user management with FastAPI ML service"
  - "deploy enterprise user management system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to work with the Enterprise User Management System, a full-stack JavaScript application that manages users, tasks, and support tickets with AI-powered analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication with JWT
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring

**Tech Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites
```bash
# Ensure you have installed:
# - Node.js (v14+)
# - Python (v3.8+)
# - MongoDB (running instance)
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
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
```

### ML Service Setup
```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

### Frontend Setup
```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Architecture Overview

```
Frontend (React:3000) → Backend (Node.js:5000) → MongoDB
                              ↓
                    ML Service (FastAPI:8000)
```

## Key API Endpoints

### Authentication
```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token, user }
```

### User Management
```javascript
// GET /api/users - Get all users (admin only)
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user (admin only)
```

### Task Management
```javascript
// GET /api/tasks - Get user tasks
// POST /api/tasks - Create task
{
  "title": "Implement user authentication",
  "description": "Add JWT-based auth",
  "priority": "high",
  "status": "todo",
  "assignedTo": "userId",
  "dueDate": "2026-05-01"
}

// PUT /api/tasks/:id - Update task status
// DELETE /api/tasks/:id - Delete task
```

### Support Tickets
```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error",
  "priority": "medium",
  "category": "technical"
}

// GET /api/tickets - Get all tickets
// PUT /api/tickets/:id/status - Update ticket status
```

### ML Analytics Endpoints
```javascript
// POST /api/ml/predict-risk - Predict user risk score
{
  "userId": "12345",
  "taskCount": 15,
  "completionRate": 0.6,
  "overdueCount": 3
}

// POST /api/ml/detect-anomaly - Detect anomalies
{
  "userId": "12345",
  "loginTime": "2026-04-15T03:00:00Z",
  "ipAddress": "192.168.1.100"
}

// POST /api/ml/detect-burnout - Analyze burnout risk
{
  "userId": "12345",
  "workingHours": 55,
  "taskLoad": 20,
  "completionRate": 0.4
}

// POST /api/ml/classify-ticket - Auto-classify support ticket
{
  "title": "Password reset issue",
  "description": "Cannot reset my password"
}
```

## Code Examples

### Frontend: User Login Component
```javascript
import React, { useState } from 'react';
import axios from 'axios';

const Login = () => {
  const [credentials, setCredentials] = useState({
    email: '',
    password: ''
  });

  const handleLogin = async (e) => {
    e.preventDefault();
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_API_URL}/api/auth/login`,
        credentials
      );
      
      // Store JWT token
      localStorage.setItem('token', response.data.token);
      localStorage.setItem('user', JSON.stringify(response.data.user));
      
      // Redirect based on role
      if (response.data.user.role === 'admin') {
        window.location.href = '/admin/dashboard';
      } else {
        window.location.href = '/user/dashboard';
      }
    } catch (error) {
      console.error('Login failed:', error.response?.data?.message);
    }
  };

  return (
    <form onSubmit={handleLogin}>
      <input
        type="email"
        placeholder="Email"
        value={credentials.email}
        onChange={(e) => setCredentials({...credentials, email: e.target.value})}
      />
      <input
        type="password"
        placeholder="Password"
        value={credentials.password}
        onChange={(e) => setCredentials({...credentials, password: e.target.value})}
      />
      <button type="submit">Login</button>
    </form>
  );
};

export default Login;
```

### Frontend: Task Kanban Board
```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    try {
      const token = localStorage.getItem('token');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      // Organize tasks by status
      const organized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'inProgress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(organized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };
  
  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      const token = localStorage.getItem('token');
      await axios.put(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks(); // Refresh board
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };
  
  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <button onClick={() => updateTaskStatus(task._id, 'inProgress')}>
              Start
            </button>
          </div>
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <button onClick={() => updateTaskStatus(task._id, 'done')}>
              Complete
            </button>
          </div>
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
          </div>
        ))}
      </div>
    </div>
  );
};

export default KanbanBoard;
```

### Backend: User Routes with JWT Authentication
```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const router = express.Router();

// Middleware to verify JWT
const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    
    if (!user) {
      return res.status(401).json({ message: 'Invalid token' });
    }
    
    req.user = user;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Authentication failed' });
  }
};

// Admin-only middleware
const adminOnly = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

// Get all users (admin only)
router.get('/users', authMiddleware, adminOnly, async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// Get current user profile
router.get('/users/me', authMiddleware, async (req, res) => {
  res.json(req.user);
});

// Update user
router.put('/users/:id', authMiddleware, async (req, res) => {
  try {
    // Users can only update themselves, admins can update anyone
    if (req.user.role !== 'admin' && req.user._id.toString() !== req.params.id) {
      return res.status(403).json({ message: 'Access denied' });
    }
    
    const updates = req.body;
    delete updates.password; // Prevent password updates via this route
    
    const user = await User.findByIdAndUpdate(
      req.params.id,
      updates,
      { new: true, runValidators: true }
    ).select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: 'Update failed', error: error.message });
  }
});

module.exports = router;
```

### Backend: ML Service Integration
```javascript
const axios = require('axios');

class MLService {
  constructor() {
    this.baseUrl = process.env.ML_SERVICE_URL || 'http://localhost:8000';
  }
  
  async predictRisk(userData) {
    try {
      const response = await axios.post(`${this.baseUrl}/api/ml/predict-risk`, {
        userId: userData.userId,
        taskCount: userData.taskCount,
        completionRate: userData.completionRate,
        overdueCount: userData.overdueCount,
        loginFrequency: userData.loginFrequency
      });
      return response.data;
    } catch (error) {
      console.error('Risk prediction failed:', error);
      throw error;
    }
  }
  
  async detectBurnout(userMetrics) {
    try {
      const response = await axios.post(`${this.baseUrl}/api/ml/detect-burnout`, {
        userId: userMetrics.userId,
        workingHours: userMetrics.workingHours,
        taskLoad: userMetrics.taskLoad,
        completionRate: userMetrics.completionRate,
        stressLevel: userMetrics.stressLevel
      });
      return response.data;
    } catch (error) {
      console.error('Burnout detection failed:', error);
      throw error;
    }
  }
  
  async classifyTicket(ticketData) {
    try {
      const response = await axios.post(`${this.baseUrl}/api/ml/classify-ticket`, {
        title: ticketData.title,
        description: ticketData.description
      });
      return response.data;
    } catch (error) {
      console.error('Ticket classification failed:', error);
      throw error;
    }
  }
  
  async detectAnomaly(activityData) {
    try {
      const response = await axios.post(`${this.baseUrl}/api/ml/detect-anomaly`, {
        userId: activityData.userId,
        loginTime: activityData.loginTime,
        ipAddress: activityData.ipAddress,
        deviceInfo: activityData.deviceInfo
      });
      return response.data;
    } catch (error) {
      console.error('Anomaly detection failed:', error);
      throw error;
    }
  }
}

module.exports = new MLService();
```

### ML Service: FastAPI Risk Prediction
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import joblib
import numpy as np
from sklearn.ensemble import RandomForestClassifier
import os

app = FastAPI()

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    completionRate: float
    overdueCount: int
    loginFrequency: Optional[int] = 0

class RiskPredictionResponse(BaseModel):
    userId: str
    riskScore: float
    riskLevel: str
    recommendations: list

# Load or initialize model
MODEL_PATH = os.getenv('MODEL_PATH', './models')
try:
    risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
except:
    # Initialize with dummy model if not exists
    risk_model = RandomForestClassifier(n_estimators=100)
    # Train with dummy data (in production, train with real data)
    X_dummy = np.random.rand(100, 4)
    y_dummy = np.random.randint(0, 2, 100)
    risk_model.fit(X_dummy, y_dummy)

@app.post("/api/ml/predict-risk", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Feature engineering
        features = np.array([[
            request.taskCount,
            request.completionRate,
            request.overdueCount,
            request.loginFrequency
        ]])
        
        # Predict risk
        risk_probability = risk_model.predict_proba(features)[0][1]
        risk_score = float(risk_probability * 100)
        
        # Determine risk level
        if risk_score < 30:
            risk_level = "low"
            recommendations = ["Keep up the good work"]
        elif risk_score < 70:
            risk_level = "medium"
            recommendations = [
                "Monitor task completion rates",
                "Consider workload redistribution"
            ]
        else:
            risk_level = "high"
            recommendations = [
                "Immediate attention required",
                "Review task assignments",
                "Consider 1-on-1 meeting"
            ]
        
        return RiskPredictionResponse(
            userId=request.userId,
            riskScore=risk_score,
            riskLevel=risk_level,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class BurnoutRequest(BaseModel):
    userId: str
    workingHours: float
    taskLoad: int
    completionRate: float
    stressLevel: Optional[float] = 0.5

class BurnoutResponse(BaseModel):
    userId: str
    burnoutScore: float
    isBurnout: bool
    suggestions: list

@app.post("/api/ml/detect-burnout", response_model=BurnoutResponse)
async def detect_burnout(request: BurnoutRequest):
    try:
        # Simple burnout calculation (can be replaced with ML model)
        hours_score = min(request.workingHours / 40, 2.0)
        task_score = min(request.taskLoad / 15, 2.0)
        completion_penalty = (1 - request.completionRate) * 2
        
        burnout_score = (hours_score + task_score + completion_penalty) / 3 * 100
        is_burnout = burnout_score > 65
        
        suggestions = []
        if is_burnout:
            suggestions = [
                "Reduce task load immediately",
                "Schedule time off",
                "Delegate tasks where possible",
                "Set up meeting with manager"
            ]
        elif burnout_score > 40:
            suggestions = [
                "Monitor workload closely",
                "Ensure regular breaks",
                "Review task priorities"
            ]
        else:
            suggestions = ["Maintain healthy work-life balance"]
        
        return BurnoutResponse(
            userId=request.userId,
            burnoutScore=float(burnout_score),
            isBurnout=is_burnout,
            suggestions=suggestions
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class TicketClassificationResponse(BaseModel):
    category: str
    priority: str
    suggestedDepartment: str
    confidence: float

@app.post("/api/ml/classify-ticket", response_model=TicketClassificationResponse)
async def classify_ticket(request: TicketClassificationRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification (replace with ML model)
        if any(word in text for word in ['password', 'login', 'access', 'account']):
            category = 'authentication'
            department = 'IT Security'
            priority = 'high'
        elif any(word in text for word in ['bug', 'error', 'crash', 'issue']):
            category = 'technical'
            department = 'Engineering'
            priority = 'medium'
        elif any(word in text for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
            department = 'Product'
            priority = 'low'
        else:
            category = 'general'
            department = 'Support'
            priority = 'medium'
        
        return TicketClassificationResponse(
            category=category,
            priority=priority,
            suggestedDepartment=department,
            confidence=0.85
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}  # Use strong secret
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
ENABLE_ONLINE_LEARNING=true
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Common Patterns

### Admin Dashboard with Analytics
```javascript
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [highRiskUsers, setHighRiskUsers] = useState([]);
  
  useEffect(() => {
    fetchAnalytics();
    fetchHighRiskUsers();
  }, []);
  
  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAnalytics(response.data);
  };
  
  const fetchHighRiskUsers = async () => {
    const token = localStorage.getItem('token');
    const usersResponse = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/users`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    // Check risk for each user
    const riskPromises = usersResponse.data.map(async (user) => {
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`,
        {
          userId: user._id,
          taskCount: user.taskCount || 0,
          completionRate: user.completionRate || 0,
          overdueCount: user.overdueCount || 0
        }
      );
      return { ...user, risk: riskResponse.data };
    });
    
    const usersWithRisk = await Promise.all(riskPromises);
    const highRisk = usersWithRisk.filter(u => u.risk.riskLevel === 'high');
    setHighRiskUsers(highRisk);
  };
  
  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="stats">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
        </div>
      )}
      
      <div className="high-risk-section">
        <h2>High Risk Users</h2>
        {highRiskUsers.map(user => (
          <div key={user._id} className="risk-card">
            <h4>{user.name}</h4>
            <p>Risk Score: {user.risk.riskScore.toFixed(1)}%</p>
            <ul>
              {user.risk.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

### Auto-Classify Support Tickets
```javascript
const createTicketWithAI = async (ticketData) => {
  try {
    const token = localStorage.getItem('token');
    
    // First, get AI classification
    const mlResponse = await axios.post(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`,
      {
        title: ticketData.title,
        description: ticketData.description
      }
    );
    
    // Create ticket with AI suggestions
    const ticketResponse = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/tickets`,
      {
        ...ticketData,
        category: mlResponse.data.category,
        priority: mlResponse.data.priority,
        assignedDepartment: mlResponse.data.suggestedDepartment,
        aiClassified: true
      },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    return ticketResponse.data;
  } catch (error) {
    console.error('Failed to create ticket:', error);
    throw error;
  }
};
```

## Troubleshooting

### JWT Token Errors
```javascript
// Clear invalid tokens
localStorage.removeItem('token');
localStorage.removeItem('user');

// Redirect to login
window.location.href = '/login';
```

### MongoDB Connection Issues
```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection string in .env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
```

### ML Service Not Responding
```bash
# Check if FastAPI is running
curl http://localhost:8000/docs

# Restart ML service
cd ml-service
uvicorn main:app --reload --port 8000

# Check logs
tail -f logs/ml-service.log
```

### CORS Errors
```javascript
// Backend: Enable CORS for frontend
const cors = require('cors');
app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### Task Status Not Updating
```javascript
// Ensure proper status values
const validStatuses = ['todo', 'inProgress', 'done'];

// Backend validation
if (!validStatuses.includes(req.body.status)) {
  return res.status(400).json({ 
    message: 'Invalid status. Must be: todo, inProgress, or done' 
  });
}
```

### Model Not Found Errors
```bash
# Create models directory
mkdir -p ml-service/models

# Train and save initial models
cd ml-service
python train_models.py
```

### Performance Issues with Large Datasets
```javascript
// Backend: Add pagination
router.get('/users', authMiddleware, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;
  
  const users = await User.find()
    .select('-password')
    .skip(skip)
    .limit(limit);
    
  const total = await User.countDocuments();
  
  res.json({
    users,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
});
```

This skill provides comprehensive coverage for working with the Enterprise User Management System, including authentication, task management, AI analytics integration, and troubleshooting common issues.
