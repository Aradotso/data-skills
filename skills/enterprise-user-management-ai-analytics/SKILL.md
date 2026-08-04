---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user management with risk detection"
  - "build admin dashboard with AI insights"
  - "add AI ticket classification system"
  - "integrate burnout detection for users"
  - "deploy user management with ML service"
  - "configure enterprise user system with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for managing enterprise users, tasks, and support tickets with integrated AI analytics. The system provides intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project delays through a FastAPI ML service, Node.js backend, and React frontend.

## Installation

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

Create `.env` file:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture Overview

The system consists of three main components:

1. **Frontend (React)** - User interface for admin and user dashboards
2. **Backend (Node.js)** - REST API for user, task, and ticket management
3. **ML Service (FastAPI)** - AI analytics endpoints for predictions and insights

## Backend API Usage

### Authentication

```javascript
// Login endpoint
const axios = require('axios');

const login = async (email, password) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email,
      password
    });
    
    const { token, user } = response.data;
    // Store token for subsequent requests
    localStorage.setItem('authToken', token);
    return user;
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
};

// Protected request with JWT
const getProfile = async () => {
  const token = localStorage.getItem('authToken');
  const response = await axios.get('http://localhost:5000/api/users/profile', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
};
```

### User Management

```javascript
// Create new user (admin only)
const createUser = async (userData) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.post(
    'http://localhost:5000/api/users',
    {
      name: userData.name,
      email: userData.email,
      role: userData.role, // 'admin' or 'user'
      department: userData.department
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get all users with filters
const getUsers = async (filters = {}) => {
  const token = localStorage.getItem('authToken');
  const params = new URLSearchParams(filters);
  const response = await axios.get(
    `http://localhost:5000/api/users?${params}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.put(
    `http://localhost:5000/api/users/${userId}`,
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.post(
    'http://localhost:5000/api/tasks',
    {
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in-progress', 'done'
      deadline: taskData.deadline
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Update task status (Kanban board)
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    { status: newStatus },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Track time on task
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    { timeSpent }, // in minutes
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('authToken');
  const response = await axios.post(
    'http://localhost:5000/api/tickets',
    {
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};

// Get user's tickets
const getMyTickets = async () => {
  const token = localStorage.getItem('authToken');
  const response = await axios.get(
    'http://localhost:5000/api/tickets/my-tickets',
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
};
```

## ML Service API Usage

### Risk Detection

```javascript
// Predict user risk score
const predictRisk = async (userId) => {
  const response = await axios.post('http://localhost:8000/api/ml/risk-prediction', {
    user_id: userId,
    features: {
      failed_login_attempts: 3,
      unusual_activity_score: 0.7,
      policy_violations: 2,
      access_pattern_anomaly: 0.6
    }
  });
  
  const { risk_score, risk_level, recommendations } = response.data;
  console.log(`Risk Score: ${risk_score}, Level: ${risk_level}`);
  return response.data;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomaly = async (userActivity) => {
  const response = await axios.post('http://localhost:8000/api/ml/anomaly-detection', {
    user_id: userActivity.userId,
    activity_data: {
      login_time: userActivity.loginTime,
      location: userActivity.location,
      actions_performed: userActivity.actions,
      data_accessed: userActivity.dataAccessed
    }
  });
  
  const { is_anomaly, anomaly_score, details } = response.data;
  if (is_anomaly) {
    console.log('Anomaly detected:', details);
  }
  return response.data;
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const analyzeBurnout = async (userId) => {
  const response = await axios.post('http://localhost:8000/api/ml/burnout-detection', {
    user_id: userId,
    workload_metrics: {
      tasks_assigned: 25,
      tasks_completed: 20,
      avg_daily_hours: 10,
      weekend_work_frequency: 0.8,
      deadline_pressure: 0.9
    }
  });
  
  const { burnout_risk, level, factors, suggestions } = response.data;
  console.log(`Burnout Risk: ${burnout_risk}%, Level: ${level}`);
  suggestions.forEach(s => console.log(`- ${s}`));
  return response.data;
};
```

### AI Ticket Classification

```javascript
// Classify and route support ticket
const classifyTicket = async (ticketText) => {
  const response = await axios.post('http://localhost:8000/api/ml/ticket-classification', {
    ticket_text: ticketText,
    metadata: {
      user_role: 'developer',
      department: 'engineering'
    }
  });
  
  const { category, priority, assigned_team, confidence } = response.data;
  console.log(`Category: ${category}, Priority: ${priority}`);
  console.log(`Route to: ${assigned_team} (confidence: ${confidence})`);
  return response.data;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await axios.post('http://localhost:8000/api/ml/project-prediction', {
    project_id: projectData.id,
    metrics: {
      tasks_total: 50,
      tasks_completed: 20,
      days_remaining: 15,
      team_size: 5,
      complexity_score: 0.8,
      dependency_count: 12
    }
  });
  
  const { delay_probability, estimated_delay_days, risk_factors } = response.data;
  console.log(`Delay Probability: ${delay_probability}%`);
  console.log(`Estimated Delay: ${estimated_delay_days} days`);
  return response.data;
};
```

## Frontend React Components

### Admin Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    try {
      const token = localStorage.getItem('authToken');
      const config = { headers: { Authorization: `Bearer ${token}` } };
      
      const [usersRes, analyticsRes] = await Promise.all([
        axios.get(`${process.env.REACT_APP_API_URL}/api/users`, config),
        axios.get(`${process.env.REACT_APP_API_URL}/api/analytics/overview`, config)
      ]);
      
      setUsers(usersRes.data);
      setAnalytics(analyticsRes.data);
      setLoading(false);
    } catch (error) {
      console.error('Failed to fetch dashboard data:', error);
    }
  };

  const handleUserRiskCheck = async (userId) => {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-prediction`,
        { user_id: userId }
      );
      alert(`Risk Score: ${response.data.risk_score}`);
    } catch (error) {
      console.error('Risk check failed:', error);
    }
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="analytics-cards">
        <div className="card">
          <h3>Total Users</h3>
          <p>{analytics.totalUsers}</p>
        </div>
        <div className="card">
          <h3>Active Tasks</h3>
          <p>{analytics.activeTasks}</p>
        </div>
        <div className="card">
          <h3>Open Tickets</h3>
          <p>{analytics.openTickets}</p>
        </div>
      </div>
      <div className="users-list">
        <h2>Users</h2>
        {users.map(user => (
          <div key={user._id} className="user-card">
            <span>{user.name} - {user.email}</span>
            <button onClick={() => handleUserRiskCheck(user._id)}>
              Check Risk
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

### Kanban Board Component

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
      const token = localStorage.getItem('authToken');
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`,
        { headers: { Authorization: `Bearer ${token}` } }
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
      const token = localStorage.getItem('authToken');
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Failed to move task:', error);
    }
  };

  const Column = ({ title, status, taskList }) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {taskList.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <select 
            value={task.status} 
            onChange={(e) => moveTask(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="in-progress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" taskList={tasks.todo} />
      <Column title="In Progress" status="inProgress" taskList={tasks.inProgress} />
      <Column title="Done" status="done" taskList={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

## Common Patterns

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'No authentication token' });
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

### ML Model Training Script

```python
# ml-service/train_models.py
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib
import os

def train_risk_model(data_path):
    """Train risk prediction model"""
    df = pd.read_csv(data_path)
    
    features = ['failed_logins', 'policy_violations', 'anomaly_score']
    X = df[features]
    y = df['risk_level']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f"Model accuracy: {accuracy:.2f}")
    
    os.makedirs('./models', exist_ok=True)
    joblib.dump(model, './models/risk_model.pkl')
    print("Model saved to ./models/risk_model.pkl")

if __name__ == "__main__":
    train_risk_model('./data/user_risk_data.csv')
```

### FastAPI ML Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List, Dict

app = FastAPI()

# Load models
risk_model = joblib.load('./models/risk_model.pkl')

class RiskPredictionRequest(BaseModel):
    user_id: str
    features: Dict[str, float]

class RiskPredictionResponse(BaseModel):
    risk_score: float
    risk_level: str
    recommendations: List[str]

@app.post("/api/ml/risk-prediction", response_model=RiskPredictionResponse)
async def predict_risk(request: RiskPredictionRequest):
    try:
        features = np.array([[
            request.features.get('failed_login_attempts', 0),
            request.features.get('policy_violations', 0),
            request.features.get('anomaly_score', 0)
        ]])
        
        risk_proba = risk_model.predict_proba(features)[0]
        risk_score = float(np.max(risk_proba))
        risk_level = ['low', 'medium', 'high'][np.argmax(risk_proba)]
        
        recommendations = []
        if risk_level == 'high':
            recommendations.append("Review user access permissions")
            recommendations.append("Enable additional authentication")
        elif risk_level == 'medium':
            recommendations.append("Monitor user activity closely")
        
        return RiskPredictionResponse(
            risk_score=risk_score,
            risk_level=risk_level,
            recommendations=recommendations
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

## Configuration

### Backend Configuration

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection failed:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Configuration

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    backend_url: str = os.getenv('BACKEND_URL', 'http://localhost:5000')
    model_path: str = os.getenv('MODEL_PATH', './models')
    log_level: str = os.getenv('LOG_LEVEL', 'INFO')
    
    class Config:
        env_file = '.env'

settings = Settings()
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const oldToken = localStorage.getItem('authToken');
    const response = await axios.post(
      'http://localhost:5000/api/auth/refresh',
      {},
      { headers: { Authorization: `Bearer ${oldToken}` } }
    );
    localStorage.setItem('authToken', response.data.token);
    return response.data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

### MongoDB Connection Issues

```javascript
// Add connection retry logic
const connectWithRetry = async () => {
  const maxRetries = 5;
  let retries = 0;
  
  while (retries < maxRetries) {
    try {
      await mongoose.connect(process.env.MONGODB_URI);
      console.log('Connected to MongoDB');
      return;
    } catch (error) {
      retries++;
      console.log(`MongoDB connection failed. Retry ${retries}/${maxRetries}`);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
  process.exit(1);
};
```

### ML Service Model Loading

```python
# Handle missing models gracefully
import os
import joblib

def load_model_safe(model_path, default=None):
    """Load model with fallback"""
    try:
        if os.path.exists(model_path):
            return joblib.load(model_path)
        else:
            print(f"Model not found: {model_path}")
            return default
    except Exception as e:
        print(f"Error loading model: {e}")
        return default

risk_model = load_model_safe('./models/risk_model.pkl')
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

### Rate Limiting for ML Endpoints

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/api/ml/risk-prediction")
@limiter.limit("10/minute")
async def predict_risk(request: Request, data: RiskPredictionRequest):
    # Implementation
    pass
```

This system provides comprehensive enterprise user management with AI-powered insights for improved decision-making and productivity.
