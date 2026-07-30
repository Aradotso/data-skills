---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and task optimization
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement risk detection and anomaly detection for users
  - create a user management dashboard with AI insights
  - build a task tracking system with burnout detection
  - set up JWT authentication for enterprise user management
  - configure AI-powered ticket classification
  - implement predictive analytics for project management
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management application with integrated AI/ML capabilities for intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics. Built with React frontend, Node.js backend, FastAPI ML service, and MongoDB database.

## What This Project Does

This system provides:
- **User & Role Management**: Secure authentication, role-based access control (RBAC)
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support Ticket System**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, user monitoring
- **User Dashboard**: Personal task overview, performance insights, notifications

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB 4.4+

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

Create `.env` file in `backend/`:
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:
```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:
```env
MODEL_PATH=./models
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
```

Start ML service:
```bash
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
# Frontend runs at http://localhost:3000
```

## Key API Endpoints

### Authentication (Backend)

```javascript
// POST /api/auth/register - Register new user
fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com',
    password: 'securePassword123',
    role: 'user'
  })
})

// POST /api/auth/login - User login
fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'securePassword123'
  })
})
.then(res => res.json())
.then(data => {
  // Store token
  localStorage.setItem('token', data.token);
})
```

### User Management (Backend)

```javascript
// GET /api/users - Get all users (Admin only)
const token = localStorage.getItem('token');
fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
  headers: {
    'Authorization': `Bearer ${token}`
  }
})

// PUT /api/users/:id - Update user
fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'John Updated',
    role: 'admin'
  })
})

// DELETE /api/users/:id - Delete user (Admin only)
fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Bearer ${token}` }
})
```

### Task Management (Backend)

```javascript
// POST /api/tasks - Create task
fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Implement authentication',
    description: 'Add JWT authentication to the system',
    assignedTo: 'userId123',
    priority: 'high',
    dueDate: '2026-05-01',
    status: 'todo'
  })
})

// GET /api/tasks/user/:userId - Get user tasks
fetch(`${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`, {
  headers: { 'Authorization': `Bearer ${token}` }
})

// PATCH /api/tasks/:id/status - Update task status
fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
  method: 'PATCH',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ status: 'in-progress' })
})
```

### Support Tickets (Backend)

```javascript
// POST /api/tickets - Create support ticket
fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    title: 'Login issue',
    description: 'Cannot login with correct credentials',
    priority: 'medium',
    category: 'technical'
  })
})

// GET /api/tickets - Get all tickets
fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
  headers: { 'Authorization': `Bearer ${token}` }
})
```

### AI Analytics Endpoints (ML Service)

```javascript
// POST /api/ml/risk-analysis - Analyze user risk
fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-analysis`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    loginAttempts: 5,
    taskCompletionRate: 0.65,
    averageResponseTime: 120,
    failedLoginCount: 2
  })
})
.then(res => res.json())
.then(data => {
  console.log('Risk Level:', data.riskLevel); // 'low', 'medium', 'high'
  console.log('Risk Score:', data.score);
})

// POST /api/ml/burnout-detection - Detect employee burnout
fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    tasksAssigned: 25,
    tasksCompleted: 18,
    averageWorkHours: 52,
    overtimeHours: 15,
    missedDeadlines: 3
  })
})
.then(res => res.json())
.then(data => {
  console.log('Burnout Risk:', data.burnoutRisk);
  console.log('Recommendations:', data.recommendations);
})

// POST /api/ml/ticket-classification - Auto-classify support ticket
fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/ticket-classification`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Cannot access dashboard',
    description: 'Getting 403 error when trying to view admin panel'
  })
})
.then(res => res.json())
.then(data => {
  console.log('Category:', data.category); // 'technical', 'access', 'feature'
  console.log('Priority:', data.priority); // 'low', 'medium', 'high', 'critical'
  console.log('Suggested Assignee:', data.assignTo);
})

// POST /api/ml/project-insights - Predictive project analytics
fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/project-insights`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: 'proj123',
    totalTasks: 50,
    completedTasks: 30,
    daysRemaining: 15,
    teamSize: 5,
    velocityTrend: [8, 7, 9, 6, 8]
  })
})
.then(res => res.json())
.then(data => {
  console.log('Completion Probability:', data.completionProbability);
  console.log('Predicted Delay (days):', data.predictedDelay);
  console.log('Recommended Actions:', data.actions);
})

// POST /api/ml/anomaly-detection - Detect unusual user behavior
fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/anomaly-detection`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: 'user123',
    loginTime: '03:00',
    loginLocation: 'unusual_country',
    dataAccessVolume: 5000,
    typicalAccessVolume: 200
  })
})
.then(res => res.json())
.then(data => {
  console.log('Is Anomaly:', data.isAnomaly);
  console.log('Anomaly Score:', data.score);
  console.log('Alert Level:', data.alertLevel);
})
```

## Common Usage Patterns

### Protected Route Component (React)

```javascript
// frontend/src/components/ProtectedRoute.js
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('role');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly && userRole !== 'admin') {
    return <Navigate to="/dashboard" />;
  }
  
  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route 
          path="/dashboard" 
          element={
            <ProtectedRoute>
              <Dashboard />
            </ProtectedRoute>
          } 
        />
        <Route 
          path="/admin" 
          element={
            <ProtectedRoute adminOnly={true}>
              <AdminPanel />
            </ProtectedRoute>
          } 
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### Task Kanban Board Component

```javascript
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    // Group tasks by status
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    fetchTasks(); // Refresh
  };

  return (
    <div className="kanban-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onMove={(id) => updateTaskStatus(id, 'in-progress')} 
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onMove={(id) => updateTaskStatus(id, 'done')} 
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
      />
    </div>
  );
};
```

### AI Risk Analysis Dashboard

```javascript
// frontend/src/components/RiskAnalytics.js
import React, { useState, useEffect } from 'react';

const RiskAnalytics = () => {
  const [userRisks, setUserRisks] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    analyzeAllUsers();
  }, []);

  const analyzeAllUsers = async () => {
    // Get all users
    const usersRes = await fetch(
      `${process.env.REACT_APP_API_URL}/api/users`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const users = await usersRes.json();

    // Analyze each user
    const risks = await Promise.all(
      users.map(async (user) => {
        const riskRes = await fetch(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-analysis`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              userId: user._id,
              loginAttempts: user.loginAttempts || 0,
              taskCompletionRate: user.taskCompletionRate || 0,
              averageResponseTime: user.avgResponseTime || 0,
              failedLoginCount: user.failedLogins || 0
            })
          }
        );
        const risk = await riskRes.json();
        return { ...user, ...risk };
      })
    );

    setUserRisks(risks.sort((a, b) => b.score - a.score));
  };

  return (
    <div className="risk-analytics">
      <h2>User Risk Analysis</h2>
      <table>
        <thead>
          <tr>
            <th>User</th>
            <th>Risk Level</th>
            <th>Risk Score</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {userRisks.map(user => (
            <tr key={user._id} className={`risk-${user.riskLevel}`}>
              <td>{user.name}</td>
              <td>{user.riskLevel}</td>
              <td>{user.score.toFixed(2)}</td>
              <td>
                {user.riskLevel === 'high' && (
                  <button onClick={() => flagUser(user._id)}>
                    Flag for Review
                  </button>
                )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};
```

### Backend Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
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

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authenticate, requireAdmin };

// Usage in routes
const express = require('express');
const router = express.Router();
const { authenticate, requireAdmin } = require('../middleware/auth');

router.get('/users', authenticate, requireAdmin, async (req, res) => {
  // Only authenticated admins can access this
  const users = await User.find();
  res.json(users);
});
```

### ML Model Training Script (Python)

```python
# ml-service/train_models.py
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib
import os

def train_risk_model():
    """Train the user risk detection model"""
    # Load training data
    data = pd.read_csv('data/user_behavior.csv')
    
    X = data[['login_attempts', 'task_completion_rate', 
              'avg_response_time', 'failed_login_count']]
    y = data['risk_level']  # 0: low, 1: medium, 2: high
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f'Risk Model Accuracy: {accuracy:.2%}')
    
    # Save model
    os.makedirs('models', exist_ok=True)
    joblib.dump(model, 'models/risk_model.pkl')

def train_burnout_model():
    """Train the employee burnout detection model"""
    data = pd.read_csv('data/employee_workload.csv')
    
    X = data[['tasks_assigned', 'tasks_completed', 'avg_work_hours',
              'overtime_hours', 'missed_deadlines']]
    y = data['burnout_risk']  # 0: no, 1: yes
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f'Burnout Model Accuracy: {accuracy:.2%}')
    
    joblib.dump(model, 'models/burnout_model.pkl')

if __name__ == '__main__':
    train_risk_model()
    train_burnout_model()
```

### FastAPI ML Service Implementation

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List
import os

app = FastAPI()

# Load models at startup
MODEL_PATH = os.getenv('MODEL_PATH', './models')
risk_model = joblib.load(f'{MODEL_PATH}/risk_model.pkl')
burnout_model = joblib.load(f'{MODEL_PATH}/burnout_model.pkl')

class RiskAnalysisRequest(BaseModel):
    userId: str
    loginAttempts: int
    taskCompletionRate: float
    averageResponseTime: int
    failedLoginCount: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    tasksAssigned: int
    tasksCompleted: int
    averageWorkHours: float
    overtimeHours: float
    missedDeadlines: int

@app.post('/api/ml/risk-analysis')
async def analyze_risk(request: RiskAnalysisRequest):
    try:
        features = np.array([[
            request.loginAttempts,
            request.taskCompletionRate,
            request.averageResponseTime,
            request.failedLoginCount
        ]])
        
        prediction = risk_model.predict(features)[0]
        probability = risk_model.predict_proba(features)[0]
        
        risk_levels = ['low', 'medium', 'high']
        
        return {
            'userId': request.userId,
            'riskLevel': risk_levels[prediction],
            'score': float(max(probability)),
            'probabilities': {
                'low': float(probability[0]),
                'medium': float(probability[1]),
                'high': float(probability[2])
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post('/api/ml/burnout-detection')
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        features = np.array([[
            request.tasksAssigned,
            request.tasksCompleted,
            request.averageWorkHours,
            request.overtimeHours,
            request.missedDeadlines
        ]])
        
        prediction = burnout_model.predict(features)[0]
        probability = burnout_model.predict_proba(features)[0][1]
        
        recommendations = []
        if probability > 0.7:
            recommendations.append('Reduce workload immediately')
            recommendations.append('Schedule wellness check-in')
        elif probability > 0.4:
            recommendations.append('Monitor work hours closely')
            recommendations.append('Consider task redistribution')
        
        return {
            'userId': request.userId,
            'burnoutRisk': 'high' if prediction == 1 else 'low',
            'probability': float(probability),
            'recommendations': recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get('/health')
async def health_check():
    return {'status': 'healthy', 'models_loaded': True}
```

## Configuration

### MongoDB Schema Examples

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  loginAttempts: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  taskCompletionRate: { type: Number, default: 0 },
  avgResponseTime: { type: Number, default: 0 },
  isActive: { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now }
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

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check token expiration
const jwt = require('jsonwebtoken');

try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  console.log('Token valid, expires:', new Date(decoded.exp * 1000));
} catch (error) {
  if (error.name === 'TokenExpiredError') {
    console.log('Token expired, please login again');
  } else {
    console.log('Invalid token:', error.message);
  }
}
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

### MongoDB Connection Issues

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

### ML Service Not Responding

Check ML service health:
```bash
curl http://localhost:8000/health
```

Verify Python dependencies:
```bash
cd ml-service
pip list | grep -E "fastapi|scikit-learn|joblib"
```

Re-train models if missing:
```bash
python train_models.py
```

### Frontend API Connection Issues

```javascript
// frontend/src/utils/api.js
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000';

export const fetchWithAuth = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  
  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': token ? `Bearer ${token}` : '',
      'Content-Type': 'application/json'
    }
  });

  if (response.status === 401) {
    localStorage.removeItem('token');
    window.location.href = '/login';
  }

  return response;
};
```

## Performance Optimization

### Caching ML Predictions

```javascript
// backend/middleware/cache.js
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 600 }); // 10 minute cache

const cacheMLPrediction = (req, res, next) => {
  const key = `ml_${req.body.userId}_${req.path}`;
  const cachedResult = cache.get(key);
  
  if (cachedResult) {
    return res.json(cachedResult);
  }
  
  res.sendResponse = res.json;
  res.json = (body) => {
    cache.set(key, body);
    res.sendResponse(body);
  };
  next();
};
```

### Database Indexing

```javascript
// Add indexes for better query performance
userSchema.index({ email: 1 });
userSchema.index({ role: 1, isActive: 1 });
taskSchema.index({ assignedTo: 1, status: 1 });
taskSchema.index({ dueDate: 1 });
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering installation, API usage, common patterns, and troubleshooting.
