---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement user task tracking with AI insights"
  - "build admin dashboard with risk detection"
  - "create ticket management system with AI classification"
  - "add anomaly detection to user management"
  - "deploy user management with ML service"
  - "configure JWT authentication for enterprise app"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It features role-based access control (Admin/User), task tracking with Kanban boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay prediction.

**Architecture:**
- Frontend: React.js (port 3000)
- Backend: Node.js with Express (port 5000)
- ML Service: FastAPI with scikit-learn and River (port 8000)
- Database: MongoDB
- Auth: JWT tokens

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
python >= 3.8
mongodb >= 4.x
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
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Setup frontend
cd ../frontend
npm install
```

### Environment Configuration

**Backend `.env`:**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/user_management
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service `.env`:**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend `.env`:**
```bash
REACT_APP_BACKEND_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

## Running the System

### Start All Services

```bash
# Terminal 1: Backend
cd backend
npm start

# Terminal 2: ML Service
cd ml-service
uvicorn main:app --reload --port 8000

# Terminal 3: Frontend
cd frontend
npm start
```

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Backend production
cd backend
npm run prod
```

## Backend API Reference

### Authentication

```javascript
// Register user (Admin only)
const axios = require('axios');

const registerUser = async (userData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'  // 'admin' or 'user'
    });
    return response.data;
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
};

// Login
const login = async (credentials) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email: credentials.email,
      password: credentials.password
    });
    // Store JWT token
    localStorage.setItem('token', response.data.token);
    return response.data;
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async (token) => {
  const response = await axios.get('http://localhost:5000/api/users', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
};

// Update user
const updateUser = async (userId, updates, token) => {
  const response = await axios.put(
    `http://localhost:5000/api/users/${userId}`,
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Delete user
const deleteUser = async (userId, token) => {
  await axios.delete(`http://localhost:5000/api/users/${userId}`, {
    headers: { Authorization: `Bearer ${token}` }
  });
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData, token) => {
  const response = await axios.post(
    'http://localhost:5000/api/tasks',
    {
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority,  // 'low', 'medium', 'high'
      status: 'todo',  // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await axios.get(
    `http://localhost:5000/api/tasks/user/${userId}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    { status },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Track time on task
const trackTime = async (taskId, timeSpent, token) => {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    { timeSpent },  // in seconds
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

### Ticket Management

```javascript
// Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await axios.post(
    'http://localhost:5000/api/tickets',
    {
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get tickets
const getTickets = async (filters, token) => {
  const params = new URLSearchParams(filters);
  const response = await axios.get(
    `http://localhost:5000/api/tickets?${params}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Update ticket status
const updateTicket = async (ticketId, updates, token) => {
  const response = await axios.patch(
    `http://localhost:5000/api/tickets/${ticketId}`,
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

## ML Service API Reference

### Risk Detection

```javascript
// Analyze user risk
const analyzeUserRisk = async (userData) => {
  const response = await axios.post('http://localhost:8000/api/ml/risk-detection', {
    userId: userData.userId,
    loginAttempts: userData.loginAttempts,
    failedLogins: userData.failedLogins,
    lastLoginTime: userData.lastLoginTime,
    activityPattern: userData.activityPattern,
    permissionChanges: userData.permissionChanges
  });
  
  // Response: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return response.data;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (behaviorData) => {
  const response = await axios.post('http://localhost:8000/api/ml/anomaly-detection', {
    userId: behaviorData.userId,
    loginTime: behaviorData.loginTime,
    loginLocation: behaviorData.loginLocation,
    deviceInfo: behaviorData.deviceInfo,
    actionsPerformed: behaviorData.actionsPerformed,
    dataAccessed: behaviorData.dataAccessed
  });
  
  // Response: { isAnomaly: true, anomalyScore: 0.85, anomalyType: 'unusual_access_pattern' }
  return response.data;
};
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
const analyzeBurnout = async (workloadData) => {
  const response = await axios.post('http://localhost:8000/api/ml/burnout-detection', {
    userId: workloadData.userId,
    tasksCompleted: workloadData.tasksCompleted,
    averageTaskTime: workloadData.averageTaskTime,
    overtimeHours: workloadData.overtimeHours,
    missedDeadlines: workloadData.missedDeadlines,
    workloadTrend: workloadData.workloadTrend,
    lastBreakDays: workloadData.lastBreakDays
  });
  
  // Response: { burnoutScore: 0.68, burnoutLevel: 'moderate', recommendations: [...] }
  return response.data;
};
```

### Ticket Classification

```javascript
// Auto-classify support ticket
const classifyTicket = async (ticketText) => {
  const response = await axios.post('http://localhost:8000/api/ml/ticket-classification', {
    subject: ticketText.subject,
    description: ticketText.description
  });
  
  // Response: { category: 'technical', priority: 'high', assignTo: 'engineering', confidence: 0.92 }
  return response.data;
};
```

### Predictive Analytics

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await axios.post('http://localhost:8000/api/ml/predict-delay', {
    projectId: projectData.projectId,
    totalTasks: projectData.totalTasks,
    completedTasks: projectData.completedTasks,
    averageCompletionRate: projectData.averageCompletionRate,
    teamSize: projectData.teamSize,
    complexity: projectData.complexity,
    remainingDays: projectData.remainingDays
  });
  
  // Response: { delayProbability: 0.73, estimatedDelay: 5, confidence: 0.88 }
  return response.data;
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_BACKEND_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (credentials) => {
    const response = await axios.post(
      `${process.env.REACT_APP_BACKEND_URL}/api/auth/login`,
      credentials
    );
    localStorage.setItem('token', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './KanbanBoard.css';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${process.env.REACT_APP_BACKEND_URL}/api/tasks/user/${userId}`
      );
      
      const categorized = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(categorized);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_BACKEND_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const renderColumn = (status, title, taskList) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      <div className="task-list">
        {taskList.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <span className={`priority ${task.priority}`}>{task.priority}</span>
            <div className="task-actions">
              {status !== 'done' && (
                <button onClick={() => moveTask(
                  task._id,
                  status === 'todo' ? 'in-progress' : 'done'
                )}>
                  Move →
                </button>
              )}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do', tasks.todo)}
      {renderColumn('in-progress', 'In Progress', tasks.inProgress)}
      {renderColumn('done', 'Done', tasks.done)}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    anomalies: [],
    burnoutAlerts: [],
    projectDelays: []
  });

  useEffect(() => {
    fetchAnalytics();
    const interval = setInterval(fetchAnalytics, 60000); // Refresh every minute
    return () => clearInterval(interval);
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [risk, anomalies, burnout, delays] = await Promise.all([
        axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-summary`),
        axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/anomalies-summary`),
        axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/burnout-summary`),
        axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/project-delays`)
      ]);

      setAnalytics({
        riskUsers: risk.data,
        anomalies: anomalies.data,
        burnoutAlerts: burnout.data,
        projectDelays: delays.data
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="analytics-section">
        <h3>High Risk Users ({analytics.riskUsers.length})</h3>
        {analytics.riskUsers.map(user => (
          <div key={user.userId} className="alert-card risk">
            <span>{user.username}</span>
            <span className="risk-score">Risk: {(user.riskScore * 100).toFixed(0)}%</span>
            <span>{user.primaryFactor}</span>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Anomalies Detected ({analytics.anomalies.length})</h3>
        {analytics.anomalies.map((anomaly, idx) => (
          <div key={idx} className="alert-card anomaly">
            <span>{anomaly.username}</span>
            <span>{anomaly.anomalyType}</span>
            <span>{new Date(anomaly.timestamp).toLocaleString()}</span>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Burnout Alerts ({analytics.burnoutAlerts.length})</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card burnout">
            <span>{alert.username}</span>
            <span className="burnout-level">{alert.burnoutLevel}</span>
            <span>Score: {(alert.burnoutScore * 100).toFixed(0)}%</span>
          </div>
        ))}
      </div>

      <div className="analytics-section">
        <h3>Project Delay Predictions</h3>
        {analytics.projectDelays.map(project => (
          <div key={project.projectId} className="alert-card delay">
            <span>{project.projectName}</span>
            <span>Delay Probability: {(project.delayProbability * 100).toFixed(0)}%</span>
            <span>Est. Delay: {project.estimatedDelay} days</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// services/api.js
import axios from 'axios';

const API_BASE = process.env.REACT_APP_BACKEND_URL;
const ML_BASE = process.env.REACT_APP_ML_SERVICE_URL;

export const userAPI = {
  getAll: () => axios.get(`${API_BASE}/api/users`),
  getById: (id) => axios.get(`${API_BASE}/api/users/${id}`),
  update: (id, data) => axios.put(`${API_BASE}/api/users/${id}`, data),
  delete: (id) => axios.delete(`${API_BASE}/api/users/${id}`)
};

export const taskAPI = {
  create: (data) => axios.post(`${API_BASE}/api/tasks`, data),
  getUserTasks: (userId) => axios.get(`${API_BASE}/api/tasks/user/${userId}`),
  updateStatus: (id, status) => axios.patch(`${API_BASE}/api/tasks/${id}/status`, { status }),
  trackTime: (id, timeSpent) => axios.post(`${API_BASE}/api/tasks/${id}/time`, { timeSpent })
};

export const ticketAPI = {
  create: (data) => axios.post(`${API_BASE}/api/tickets`, data),
  getAll: (filters) => axios.get(`${API_BASE}/api/tickets`, { params: filters }),
  update: (id, data) => axios.patch(`${API_BASE}/api/tickets/${id}`, data)
};

export const mlAPI = {
  analyzeRisk: (data) => axios.post(`${ML_BASE}/api/ml/risk-detection`, data),
  detectAnomaly: (data) => axios.post(`${ML_BASE}/api/ml/anomaly-detection`, data),
  analyzeBurnout: (data) => axios.post(`${ML_BASE}/api/ml/burnout-detection`, data),
  classifyTicket: (data) => axios.post(`${ML_BASE}/api/ml/ticket-classification`, data),
  predictDelay: (data) => axios.post(`${ML_BASE}/api/ml/predict-delay`, data)
};
```

## ML Model Training

### Custom Risk Detection Model

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import numpy as np

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.scaler = StandardScaler()
        self.model = RandomForestClassifier(n_estimators=100, random_state=42)
        
    def prepare_features(self, user_data):
        """Extract features from user data"""
        features = [
            user_data.get('loginAttempts', 0),
            user_data.get('failedLogins', 0),
            user_data.get('permissionChanges', 0),
            user_data.get('dataAccessCount', 0),
            user_data.get('offHoursActivity', 0),
            user_data.get('suspiciousPatterns', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def train(self, training_data, labels):
        """Train the risk detection model"""
        X = self.scaler.fit_transform(training_data)
        self.model.fit(X, labels)
        joblib.dump(self.model, self.model_path)
        joblib.dump(self.scaler, f"{self.model_path}.scaler")
    
    def predict_risk(self, user_data):
        """Predict risk score for a user"""
        features = self.prepare_features(user_data)
        scaled_features = self.scaler.transform(features)
        
        risk_score = self.model.predict_proba(scaled_features)[0][1]
        
        # Determine risk level
        if risk_score >= 0.7:
            level = 'high'
        elif risk_score >= 0.4:
            level = 'medium'
        else:
            level = 'low'
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': level,
            'factors': self._identify_risk_factors(user_data, features)
        }
    
    def _identify_risk_factors(self, user_data, features):
        """Identify primary risk factors"""
        factors = []
        if user_data.get('failedLogins', 0) > 3:
            factors.append('Multiple failed login attempts')
        if user_data.get('offHoursActivity', 0) > 5:
            factors.append('Unusual off-hours activity')
        if user_data.get('permissionChanges', 0) > 0:
            factors.append('Recent permission changes')
        return factors
```

### FastAPI ML Endpoint

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_detector import RiskDetector
from models.anomaly_detector import AnomalyDetector
from models.burnout_analyzer import BurnoutAnalyzer

app = FastAPI()

risk_detector = RiskDetector()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

class UserRiskRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    lastLoginTime: str
    activityPattern: dict
    permissionChanges: int

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    loginLocation: str
    deviceInfo: str
    actionsPerformed: list
    dataAccessed: list

class BurnoutRequest(BaseModel):
    userId: str
    tasksCompleted: int
    averageTaskTime: float
    overtimeHours: float
    missedDeadlines: int
    workloadTrend: list
    lastBreakDays: int

@app.post("/api/ml/risk-detection")
async def detect_risk(request: UserRiskRequest):
    try:
        result = risk_detector.predict_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyRequest):
    try:
        result = anomaly_detector.detect(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def analyze_burnout(request: BurnoutRequest):
    try:
        result = burnout_analyzer.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Troubleshooting

### JWT Authentication Issues

```javascript
// Check token validity
const verifyToken = () => {
  const token = localStorage.getItem('token');
  if (!token) {
    console.error('No token found');
    return false;
  }
  
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const isExpired = payload.exp * 1000 < Date.now();
    
    if (isExpired) {
      console.error('Token expired');
      localStorage.removeItem('token');
      return false;
    }
    
    return true;
  } catch (error) {
    console.error('Invalid token format');
    return false;
  }
};
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected, attempting to reconnect...');
  connectDB();
});

module.exports = connectDB;
```

### ML Service Not Responding

```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Test ML endpoint
curl -X POST http://localhost:8000/api/ml/risk-detection \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "test",
    "loginAttempts": 5,
    "failedLogins": 2,
    "lastLoginTime": "2026-01-01T10:00:00Z",
    "activityPattern": {},
    "permissionChanges": 0
  }'
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Model Performance Issues

```python
# ml-service/utils/model_monitor.py
import time
from functools import wraps

def monitor_performance(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start_time = time.time()
        result = await func(*args, **kwargs)
        elapsed = time.time() - start_time
        
        if elapsed > 1.0:  # Log if prediction takes > 1 second
            print(f"WARNING: {func.__name__} took {elapsed:.2f}s")
        
        return result
    return wrapper

# Use decorator on ML endpoints
@app.post("/api/ml/risk-detection")
@monitor_performance
async def detect_risk(request: UserRiskRequest):
    # ... implementation
    pass
```

## Deployment

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      MONGODB_URI: mongodb://mongodb:27017/user_management
      JWT_SECRET: ${JWT_SECRET}
      ML_SERVICE_URL: http://ml-service:8000
    depends_on:
      - mongodb
