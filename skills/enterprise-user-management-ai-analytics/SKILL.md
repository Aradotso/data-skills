---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with AI insights"
  - "integrate ML-based ticket classification system"
  - "build admin dashboard with risk detection"
  - "configure user management with AI features"
  - "add burnout detection to user system"
  - "implement anomaly detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack JavaScript/Node.js application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities for predictive analytics, risk detection, anomaly detection, and burnout analysis.

## What It Does

This system provides:
- **User Management**: CRUD operations with role-based access control (RBAC)
- **Task Management**: Kanban board with time tracking and status updates
- **Ticket System**: Support ticket creation and AI-powered classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Centralized monitoring with audit logs and alerts
- **User Dashboard**: Personal task tracking and performance insights

## Installation

### Prerequisites
```bash
# Required software
node --version  # v14 or higher
npm --version   # v6 or higher
python --version  # Python 3.8+ for ML service
mongod --version  # MongoDB 4.4+
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env  # Configure environment variables

# Setup ML service
cd ../ml-service
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env)**:
```bash
# Server
PORT=5000
NODE_ENV=development

# MongoDB
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT Authentication
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRES_IN=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
FRONTEND_URL=http://localhost:3000
```

**ML Service (.env)**:
```bash
# ML Service Port
ML_PORT=8000

# MongoDB Connection
MONGO_URI=mongodb://localhost:27017/enterprise_user_mgmt

# Model Paths
MODEL_DIR=./models
RISK_MODEL_PATH=./models/risk_model.pkl
ANOMALY_MODEL_PATH=./models/anomaly_model.pkl
```

## Running the System

### Start All Services

```bash
# Terminal 1 - Backend
cd backend
npm start
# Runs at http://localhost:5000

# Terminal 2 - ML Service
cd ml-service
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000

# Terminal 3 - Frontend
cd frontend
npm start
# Runs at http://localhost:3000
```

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Start backend with production mode
cd backend
NODE_ENV=production npm start
```

## API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register - Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login - User login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management

```javascript
// GET /api/users - Get all users (Admin only)
const fetchAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, userData, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// POST /api/tasks - Create new task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'inprogress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'inprogress', 'done'
  });
  return response.json();
};
```

### Support Tickets

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (filters available)
const getTickets = async (filters, token) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### AI/ML Analytics Endpoints

```javascript
// POST /api/ml/risk-prediction - Predict user risk score
const predictRisk = async (userId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/risk-prediction', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ userId })
  });
  return response.json();
  // Returns: { userId, riskScore, riskLevel, factors }
};

// POST /api/ml/anomaly-detection - Detect anomalous behavior
const detectAnomaly = async (userActivity, token) => {
  const response = await fetch('http://localhost:5000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      activityType: userActivity.activityType,
      location: userActivity.location
    })
  });
  return response.json();
  // Returns: { isAnomaly, anomalyScore, alertLevel }
};

// POST /api/ml/burnout-analysis - Analyze employee burnout risk
const analyzeBurnout = async (userId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ userId })
  });
  return response.json();
  // Returns: { burnoutScore, workloadLevel, recommendations }
};

// POST /api/ml/ticket-classification - AI-powered ticket routing
const classifyTicket = async (ticketId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/ticket-classification', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ ticketId })
  });
  return response.json();
  // Returns: { category, priority, assignedDepartment }
};

// POST /api/ml/project-insights - Predict project delays
const getProjectInsights = async (projectId, token) => {
  const response = await fetch('http://localhost:5000/api/ml/project-insights', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ projectId })
  });
  return response.json();
  // Returns: { delayProbability, estimatedCompletion, risks }
};
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      // Verify token and fetch user data
      fetchUserProfile(token).then(setUser);
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inprogress: data.filter(t => t.status === 'inprogress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks(); // Refresh
  };

  return (
    <div className="kanban-board">
      {['todo', 'inprogress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={status} 
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inprogress">In Progress</option>
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

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = ({ token }) => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    anomalies: [],
    burnoutAlerts: []
  });

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    // Fetch risk predictions for all users
    const riskResponse = await fetch('http://localhost:5000/api/ml/risk-dashboard', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const riskData = await riskResponse.json();

    // Fetch recent anomalies
    const anomalyResponse = await fetch('http://localhost:5000/api/ml/anomalies/recent', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const anomalyData = await anomalyResponse.json();

    // Fetch burnout alerts
    const burnoutResponse = await fetch('http://localhost:5000/api/ml/burnout-alerts', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskUsers: riskData.highRiskUsers,
      anomalies: anomalyData.anomalies,
      burnoutAlerts: burnoutData.alerts
    });
  };

  return (
    <div className="ai-analytics-dashboard">
      <section className="risk-section">
        <h2>High Risk Users</h2>
        {analytics.riskUsers.map(user => (
          <div key={user.userId} className="alert-card">
            <p>{user.username} - Risk Score: {user.riskScore.toFixed(2)}</p>
            <span className={`badge ${user.riskLevel}`}>{user.riskLevel}</span>
          </div>
        ))}
      </section>

      <section className="anomaly-section">
        <h2>Recent Anomalies</h2>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="alert-card">
            <p>{anomaly.description}</p>
            <small>{new Date(anomaly.timestamp).toLocaleString()}</small>
          </div>
        ))}
      </section>

      <section className="burnout-section">
        <h2>Burnout Alerts</h2>
        {analytics.burnoutAlerts.map((alert, idx) => (
          <div key={idx} className="alert-card">
            <p>{alert.username} - Score: {alert.burnoutScore}</p>
            <p className="recommendation">{alert.recommendation}</p>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Backend API Implementation Patterns

### User Controller Example

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get all users (Admin only)
exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json({ success: true, users });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Update user
exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    // Remove sensitive fields
    delete updates.password;
    delete updates.role; // Only admins can change roles
    
    const user = await User.findByIdAndUpdate(id, updates, { new: true }).select('-password');
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    
    res.json({ success: true, user });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};

// Delete user
exports.deleteUser = async (req, res) => {
  try {
    const { id } = req.params;
    const user = await User.findByIdAndDelete(id);
    
    if (!user) {
      return res.status(404).json({ success: false, message: 'User not found' });
    }
    
    res.json({ success: true, message: 'User deleted successfully' });
  } catch (error) {
    res.status(500).json({ success: false, message: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  try {
    let token;
    
    if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
      token = req.headers.authorization.split(' ')[1];
    }
    
    if (!token) {
      return res.status(401).json({ success: false, message: 'Not authorized' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    
    next();
  } catch (error) {
    res.status(401).json({ success: false, message: 'Invalid token' });
  }
};

// Admin only middleware
exports.adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ success: false, message: 'Admin access required' });
  }
};
```

### Task Model

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
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['todo', 'inprogress', 'done'],
    default: 'todo'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high'],
    default: 'medium'
  },
  dueDate: {
    type: Date
  },
  timeSpent: {
    type: Number,
    default: 0 // in minutes
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}, {
  timestamps: true
});

module.exports = mongoose.model('Task', taskSchema);
```

## ML Service Implementation

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from datetime import datetime

app = FastAPI(title="Enterprise ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models (implement model training separately)
# risk_model = joblib.load('models/risk_model.pkl')
# anomaly_model = joblib.load('models/anomaly_model.pkl')

class RiskPredictionRequest(BaseModel):
    userId: str
    taskCount: int
    completionRate: float
    avgResponseTime: float
    missedDeadlines: int

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    activityType: str
    location: str
    activityCount: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    hoursWorked: float
    tasksCompleted: int
    overtimeHours: float
    daysOff: int

@app.post("/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior patterns"""
    try:
        # Feature engineering
        features = np.array([[
            request.taskCount,
            request.completionRate,
            request.avgResponseTime,
            request.missedDeadlines
        ]])
        
        # Mock prediction (replace with actual model)
        risk_score = (request.missedDeadlines * 0.4 + 
                     (1 - request.completionRate) * 0.6)
        
        risk_level = 'low' if risk_score < 0.3 else 'medium' if risk_score < 0.7 else 'high'
        
        return {
            "userId": request.userId,
            "riskScore": float(risk_score),
            "riskLevel": risk_level,
            "factors": {
                "missedDeadlines": request.missedDeadlines,
                "completionRate": request.completionRate
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalous user behavior"""
    try:
        # Parse login time
        login_hour = datetime.fromisoformat(request.loginTime).hour
        
        # Simple rule-based anomaly detection
        is_anomaly = False
        anomaly_reasons = []
        
        if login_hour < 6 or login_hour > 22:
            is_anomaly = True
            anomaly_reasons.append("Unusual login time")
        
        if request.activityCount > 100:
            is_anomaly = True
            anomaly_reasons.append("Excessive activity count")
        
        anomaly_score = len(anomaly_reasons) / 2.0
        
        return {
            "userId": request.userId,
            "isAnomaly": is_anomaly,
            "anomalyScore": anomaly_score,
            "reasons": anomaly_reasons,
            "alertLevel": "high" if is_anomaly else "normal"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/analyze-burnout")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    """Analyze employee burnout risk"""
    try:
        # Calculate burnout score
        overtime_factor = min(request.overtimeHours / 40, 1.0)
        rest_factor = max(0, 1 - (request.daysOff / 30))
        workload_factor = min(request.hoursWorked / 60, 1.0)
        
        burnout_score = (overtime_factor * 0.4 + 
                        rest_factor * 0.3 + 
                        workload_factor * 0.3)
        
        recommendations = []
        if burnout_score > 0.7:
            recommendations.append("Reduce overtime hours")
            recommendations.append("Schedule time off")
        elif burnout_score > 0.5:
            recommendations.append("Monitor workload closely")
        
        return {
            "userId": request.userId,
            "burnoutScore": float(burnout_score),
            "workloadLevel": "high" if burnout_score > 0.7 else "moderate" if burnout_score > 0.5 else "normal",
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Troubleshooting

### Common Issues

**MongoDB Connection Errors**
```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string in .env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

**JWT Token Errors**
```javascript
// Ensure JWT_SECRET is set
console.log(process.env.JWT_SECRET); // Should not be undefined

// Clear old tokens
localStorage.removeItem('token');

// Re-login to get fresh token
```

**CORS Issues**
```javascript
// backend/server.js - Ensure CORS is configured
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**ML Service Not Responding**
```bash
# Check if service is running
curl http://localhost:8000/health

# Restart ML service
cd ml-service
uvicorn main:app --reload --port 8000

# Check Python dependencies
pip list | grep fastapi
```

**Port Already in Use**
```bash
# Find process using port
lsof -i :5000  # Backend
lsof -i :3000  # Frontend
lsof -i :8000  # ML Service

# Kill process
kill -9 <PID>
```

### Database Issues

```bash
# Reset database
mongo enterprise_user_mgmt --eval "db.dropDatabase()"

# Create indexes for performance
mongo enterprise_user_mgmt --eval "db.users.createIndex({email: 1}, {unique: true})"
mongo enterprise_user_mgmt --eval "db.tasks.createIndex({assignedTo: 1, status: 1})"
```

### Development Tips

```javascript
// Enable detailed logging
// backend/server.js
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path}`, req.body);
  next();
});

// Add request timeout
const timeout = require('connect-timeout');
app.use(timeout('30s'));
```

## Testing

### API Testing with cURL

```bash
# Register user
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"password123"}'

# Login
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Get tasks (with token)
curl -X GET http://localhost:5000/api/tasks/user/USER_ID \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# Test ML service
curl -X POST http://localhost:8000/predict-risk \
  -H "Content-Type: application/json" \
  -d '{"userId":"123","taskCount":10,"completionRate":0.8,"avgResponseTime":120,"missedDeadlines":2}'
```

This system provides a complete enterprise user management solution with AI-powered analytics for modern organizations.
