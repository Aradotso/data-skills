---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task management, ticket routing, and behavioral insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with task tracking"
  - "build admin dashboard with AI insights"
  - "integrate ML-based ticket classification"
  - "add burnout detection to user system"
  - "configure AI analytics for enterprise app"
  - "deploy user management with FastAPI ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with machine learning capabilities. The system provides:

- **User Management**: Role-based access control (Admin/User), authentication via JWT
- **Task Tracking**: Kanban board workflow (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Smart ticket routing and classification
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Real-time Insights**: Behavioral analytics and performance monitoring

The architecture consists of three services:
1. **Frontend** (React.js on port 3000)
2. **Backend** (Node.js REST API on port 5000)
3. **ML Service** (FastAPI on port 8000)

## Installation

### Prerequisites
```bash
# Node.js 14+ and Python 3.8+ required
node --version
python --version
```

### Full Stack Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend `.env`:**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service `.env`:**
```bash
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=info
```

**Frontend `.env`:**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Services

### Start Backend
```bash
cd backend
npm start
# Runs on http://localhost:5000
```

### Start ML Service
```bash
cd ml-service
uvicorn main:app --reload
# Runs on http://localhost:8000
```

### Start Frontend
```bash
cd frontend
npm start
# Runs on http://localhost:3000
```

### Production Build
```bash
# Frontend production build
cd frontend
npm run build

# Backend production
cd backend
NODE_ENV=production npm start

# ML service with workers
cd ml-service
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const response = await fetch('http://localhost:5000/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    username: 'john.doe',
    email: 'john@example.com',
    password: 'SecurePass123',
    role: 'user'
  })
});

// Login
const loginResponse = await fetch('http://localhost:5000/api/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'john@example.com',
    password: 'SecurePass123'
  })
});
const { token, user } = await loginResponse.json();

// Authenticated requests
const headers = {
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${token}`
};
```

### User Management (Admin)

```javascript
// Get all users
const users = await fetch('http://localhost:5000/api/users', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(res => res.json());

// Create user
await fetch('http://localhost:5000/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    username: 'jane.smith',
    email: 'jane@example.com',
    role: 'user',
    department: 'Engineering'
  })
});

// Update user
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    role: 'admin',
    status: 'active'
  })
});

// Delete user
await fetch(`http://localhost:5000/api/users/${userId}`, {
  method: 'DELETE',
  headers: { 'Authorization': `Bearer ${token}` }
});
```

### Task Management

```javascript
// Create task
const task = await fetch('http://localhost:5000/api/tasks', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    title: 'Implement user authentication',
    description: 'Add JWT-based authentication',
    assignedTo: userId,
    priority: 'high',
    status: 'todo',
    dueDate: '2026-05-01'
  })
}).then(res => res.json());

// Update task status
await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    status: 'in-progress'
  })
});

// Get user tasks
const myTasks = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(res => res.json());

// Track time on task
await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    duration: 3600, // seconds
    date: new Date().toISOString()
  })
});
```

### Support Tickets

```javascript
// Create support ticket
const ticket = await fetch('http://localhost:5000/api/tickets', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    title: 'Cannot access dashboard',
    description: 'Getting 403 error when accessing admin dashboard',
    priority: 'medium',
    category: 'technical'
  })
}).then(res => res.json());

// Get all tickets (admin)
const tickets = await fetch('http://localhost:5000/api/tickets', {
  headers: { 'Authorization': `Bearer ${token}` }
}).then(res => res.json());

// Update ticket
await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    status: 'in-progress',
    assignedTo: adminId
  })
});
```

## ML Service API Usage

### Risk Prediction

```javascript
// Predict user risk based on behavior
const riskAnalysis = await fetch('http://localhost:8000/api/ml/risk-prediction', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    loginFrequency: 45,
    taskCompletionRate: 0.65,
    avgResponseTime: 24.5,
    ticketCount: 8,
    failedLoginAttempts: 2
  })
}).then(res => res.json());

// Response: { riskScore: 0.35, riskLevel: 'medium', factors: [...] }
```

### Anomaly Detection

```javascript
// Detect anomalous user behavior
const anomalyCheck = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    loginTime: '03:45:00',
    loginLocation: 'Unknown',
    activityPattern: [0, 0, 0, 15, 0, 0, 0],
    dataAccessVolume: 1500
  })
}).then(res => res.json());

// Response: { isAnomaly: true, anomalyScore: 0.87, reason: 'Unusual login time' }
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
const burnoutAnalysis = await fetch('http://localhost:8000/api/ml/burnout-detection', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    userId: userId,
    workHoursPerWeek: 55,
    overtimeHours: 15,
    taskLoad: 18,
    completionRate: 0.70,
    stressIndicators: {
      missedDeadlines: 3,
      lateSubmissions: 5,
      ticketEscalations: 2
    }
  })
}).then(res => res.json());

// Response: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
```

### Ticket Classification

```javascript
// Auto-classify support ticket
const classification = await fetch('http://localhost:8000/api/ml/classify-ticket', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    title: 'Password reset not working',
    description: 'I cannot reset my password using the forgot password link',
    userHistory: {
      previousTickets: 2,
      avgResolutionTime: 48
    }
  })
}).then(res => res.json());

// Response: { category: 'authentication', priority: 'high', suggestedAssignee: 'security-team' }
```

### Project Delay Prediction

```javascript
// Predict project delays
const delayPrediction = await fetch('http://localhost:8000/api/ml/predict-delay', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    projectId: projectId,
    plannedDuration: 30,
    completedTasks: 12,
    totalTasks: 45,
    teamSize: 5,
    currentDay: 15,
    blockers: 3
  })
}).then(res => res.json());

// Response: { delayProbability: 0.65, estimatedDelay: 7, recommendations: [...] }
```

## React Frontend Patterns

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [user, setUser] = useState(null);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserData();
    fetchTasks();
  }, []);

  const fetchUserData = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/user/profile`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setUser(response.data);
    } catch (error) {
      console.error('Failed to fetch user:', error);
    }
  };

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setTasks(response.data);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>Welcome, {user?.username}</h1>
      <div className="kanban-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
          ))}
        </div>
        <div className="column">
          <h2>In Progress</h2>
          {tasks.filter(t => t.status === 'in-progress').map(task => (
            <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
          ))}
        </div>
        <div className="column">
          <h2>Done</h2>
          {tasks.filter(t => t.status === 'done').map(task => (
            <TaskCard key={task._id} task={task} onStatusChange={updateTaskStatus} />
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component with AI

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
    fetchRiskAnalysis();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setAnalytics(response.data);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  const fetchRiskAnalysis = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/high-risk-users`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      setRiskUsers(response.data);
    } catch (error) {
      console.error('Failed to fetch risk analysis:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <h1>AI-Powered Analytics</h1>
      
      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks || 0}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets || 0}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h2>High Risk Users</h2>
        {riskUsers.map(user => (
          <div key={user.userId} className="risk-card">
            <h4>{user.username}</h4>
            <p>Risk Score: {(user.riskScore * 100).toFixed(1)}%</p>
            <p>Reason: {user.reason}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

### Protected Route Component

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" replace />;
  }

  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" replace />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import ProtectedRoute from './components/ProtectedRoute';
import AdminDashboard from './pages/AdminDashboard';
import UserDashboard from './pages/UserDashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute requireAdmin={true}>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

## Python ML Service Implementation

### FastAPI Main Application

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Optional
import logging

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Models (load pre-trained models)
risk_model = None
anomaly_model = None
burnout_model = None

@app.on_event("startup")
async def load_models():
    global risk_model, anomaly_model, burnout_model
    try:
        risk_model = joblib.load('./models/risk_model.pkl')
        anomaly_model = joblib.load('./models/anomaly_model.pkl')
        burnout_model = joblib.load('./models/burnout_model.pkl')
        logging.info("Models loaded successfully")
    except Exception as e:
        logging.error(f"Failed to load models: {e}")

# Pydantic models
class RiskPredictionRequest(BaseModel):
    userId: str
    loginFrequency: int
    taskCompletionRate: float
    avgResponseTime: float
    ticketCount: int
    failedLoginAttempts: int

class RiskPredictionResponse(BaseModel):
    riskScore: float
    riskLevel: str
    factors: List[str]

class AnomalyDetectionRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    activityPattern: List[int]
    dataAccessVolume: int

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workHoursPerWeek: float
    overtimeHours: float
    taskLoad: int
    completionRate: float
    stressIndicators: dict

# Endpoints
@app.post("/api/ml/risk-prediction", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        # Feature engineering
        features = np.array([[
            request.loginFrequency,
            request.taskCompletionRate,
            request.avgResponseTime,
            request.ticketCount,
            request.failedLoginAttempts
        ]])
        
        # Predict
        risk_score = risk_model.predict_proba(features)[0][1]
        
        # Determine risk level
        if risk_score < 0.3:
            risk_level = "low"
        elif risk_score < 0.7:
            risk_level = "medium"
        else:
            risk_level = "high"
        
        # Identify key factors
        factors = []
        if request.failedLoginAttempts > 3:
            factors.append("High failed login attempts")
        if request.taskCompletionRate < 0.6:
            factors.append("Low task completion rate")
        if request.ticketCount > 10:
            factors.append("High ticket volume")
        
        return RiskPredictionResponse(
            riskScore=float(risk_score),
            riskLevel=risk_level,
            factors=factors
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        # Parse login time to hour
        hour = int(request.loginTime.split(':')[0])
        
        # Feature engineering
        features = np.array([[
            hour,
            1 if request.loginLocation == "Unknown" else 0,
            sum(request.activityPattern),
            request.dataAccessVolume
        ]])
        
        # Predict anomaly
        is_anomaly = anomaly_model.predict(features)[0] == -1
        anomaly_score = anomaly_model.score_samples(features)[0]
        
        # Determine reason
        reason = []
        if hour < 6 or hour > 22:
            reason.append("Unusual login time")
        if request.loginLocation == "Unknown":
            reason.append("Unknown location")
        if request.dataAccessVolume > 1000:
            reason.append("High data access volume")
        
        return {
            "isAnomaly": bool(is_anomaly),
            "anomalyScore": float(abs(anomaly_score)),
            "reason": ", ".join(reason) if reason else "Normal behavior"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        # Feature engineering
        stress_score = (
            request.stressIndicators.get('missedDeadlines', 0) +
            request.stressIndicators.get('lateSubmissions', 0) +
            request.stressIndicators.get('ticketEscalations', 0)
        )
        
        features = np.array([[
            request.workHoursPerWeek,
            request.overtimeHours,
            request.taskLoad,
            request.completionRate,
            stress_score
        ]])
        
        # Predict burnout risk
        burnout_score = burnout_model.predict_proba(features)[0][1]
        
        # Determine risk level
        if burnout_score < 0.3:
            risk = "low"
        elif burnout_score < 0.7:
            risk = "medium"
        else:
            risk = "high"
        
        # Generate recommendations
        recommendations = []
        if request.overtimeHours > 10:
            recommendations.append("Reduce overtime hours")
        if request.taskLoad > 15:
            recommendations.append("Redistribute workload")
        if request.completionRate < 0.7:
            recommendations.append("Provide additional support")
        
        return {
            "burnoutRisk": risk,
            "score": float(burnout_score),
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: dict):
    try:
        text = f"{request['title']} {request['description']}"
        
        # Simple keyword-based classification
        categories = {
            'authentication': ['password', 'login', 'access', 'credentials'],
            'technical': ['error', 'bug', 'crash', 'issue'],
            'feature': ['request', 'enhancement', 'add', 'new'],
            'performance': ['slow', 'timeout', 'delay', 'loading']
        }
        
        text_lower = text.lower()
        detected_category = 'general'
        
        for category, keywords in categories.items():
            if any(keyword in text_lower for keyword in keywords):
                detected_category = category
                break
        
        # Determine priority
        priority = 'medium'
        if 'urgent' in text_lower or 'critical' in text_lower:
            priority = 'high'
        elif 'minor' in text_lower:
            priority = 'low'
        
        return {
            "category": detected_category,
            "priority": priority,
            "suggestedAssignee": f"{detected_category}-team"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": risk_model is not None}
```

## Common Patterns

### Authentication Flow

```javascript
// auth.js - Centralized auth utilities
export const login = async (email, password) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  
  if (!response.ok) throw new Error('Login failed');
  
  const { token, user } = await response.json();
  localStorage.setItem('token', token);
  localStorage.setItem('user', JSON.stringify(user));
  
  return { token, user };
};

export const logout = () => {
  localStorage.removeItem('token');
  localStorage.removeItem('user');
  window.location.href = '/login';
};

export const getAuthHeaders = () => ({
  'Content-Type': 'application/json',
  'Authorization': `Bearer ${localStorage.getItem('token')}`
});
```

### API Client Setup

```javascript
// api/client.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000
});

// Request interceptor
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.clear();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Real-time Task Updates

```javascript
// hooks/useRealTimeUpdates.js
import { useEffect, useState } from 'react';
import io from 'socket.io-client';

export const useRealTimeUpdates = (userId) => {
  const [notifications, setNotifications] = useState([]);
  
  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL);
    
    socket.on('connect', () => {
      socket.emit('join', userId);
    });
    
    socket.on('task-update', (data) => {
      setNotifications(prev => [...prev, data]);
    });
    
    socket.on('ticket-assigned', (data) => {
      setNotifications(prev => [...prev, data]);
    });
    
    return () => socket.disconnect();
  }, [userId]);
  
  return notifications;
};
```

## Troubleshooting

### Backend Connection Issues

```bash
# Check if MongoDB is running
mongod --version
# Start MongoDB if needed
mongod --dbpath /path/to/data

# Verify environment variables
cd backend
cat .env

# Test API endpoint
curl http://localhost:5000/health
```

### ML Service Not Responding

```python
# Check if models exist
import os
os.listdir('./ml-service/models')

# Test ML service
curl http://localhost:8000/health

# Check Python dependencies
cd ml-service
pip list | grep -E "fastapi|scikit-learn|river"

# Run with verbose logging
uvicorn main:app --reload --log-level debug
```

### Authentication Errors

```javascript
// Clear local storage and re-login
localStorage.clear();

// Verify token is valid
const token = localStorage.getItem('token');
console.log('Token:', token ? 'exists' : 'missing');

// Check token expiration
import jwt_decode from 'jwt-decode';
const decoded = jwt_decode(token);
console.log('Expires:', new Date(decoded.exp * 1000));
```

### CORS Issues

```javascript
// Backend: Verify CORS configuration
app.use(cors({
  origin: ['http://localhost:3000'],
  credentials: true
}));

// Frontend: Include credentials
fetch(url, {
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' }
});
```

### Model Loading Failures

```python
# Re-train and save models
from sklearn.ensemble import RandomForestClassifier
import joblib

# Example: Train risk model
X_train = [[45, 0.65, 24.5, 8, 2]]  # Sample data
y_train = [0]
model = RandomForestClassifier()
model.fit(X_train, y_train)
joblib.dump(model, './models/risk_model.pkl')

# Verify model file
import os
print(os.path.exists('./models/risk_model.pkl'))
```

### Database Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNew
