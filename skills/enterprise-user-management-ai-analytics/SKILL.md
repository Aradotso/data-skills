---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task management built with React, Node.js, and FastAPI
triggers:
  - how do I set up the enterprise user management system
  - help me integrate AI analytics for user management
  - show me how to implement risk detection and anomaly detection
  - guide me through the user management API endpoints
  - how do I configure the ML service for burnout detection
  - help me build task management with Kanban board
  - show me JWT authentication implementation
  - how do I use the AI assistant features
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with a comprehensive enterprise user management system that combines traditional CRUD operations with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System provides:
- **User Management**: Role-based access control, authentication, and user CRUD operations
- **Task Management**: Kanban-style task boards with time tracking and progress monitoring
- **Support Ticketing**: Intelligent ticket classification and routing using AI
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, and project insights
- **Admin Dashboard**: Comprehensive analytics and audit logging

## Architecture Overview

The system consists of three main components:
1. **Frontend**: React.js application (port 3000)
2. **Backend**: Node.js REST API with MongoDB (port 5000)
3. **ML Service**: FastAPI-based AI/ML microservice (port 8000)

## Installation & Setup

### Prerequisites

```bash
# Required tools
- Node.js v14+
- Python 3.8+
- MongoDB instance
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
NODE_ENV=development
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

## Backend API Endpoints

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "name": "John Doe",
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

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "userId",
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
Headers: { "Authorization": "Bearer ${token}" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer ${token}" }
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: { "Authorization": "Bearer ${token}" }
{
  "name": "Jane Smith Updated",
  "role": "manager"
}

// Delete user
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer ${token}" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer ${token}" }
{
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PUT /api/tasks/:taskId
{
  "status": "in-progress"
}

// Track time on task
POST /api/tasks/:taskId/time
{
  "duration": 3600, // seconds
  "date": "2026-04-20"
}

// Get user tasks
GET /api/tasks/user/:userId
Headers: { "Authorization": "Bearer ${token}" }
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer ${token}" }
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard",
  "priority": "medium",
  "category": "technical"
}

// Get user tickets
GET /api/tickets/user/:userId
Headers: { "Authorization": "Bearer ${token}" }

// Update ticket (Admin)
PUT /api/tickets/:ticketId
Headers: { "Authorization": "Bearer ${token}" }
{
  "status": "in-progress",
  "assignedTo": "supportUserId"
}
```

## ML Service API Endpoints

### Risk Prediction

```javascript
// Predict user risk level
POST /api/ml/predict-risk
{
  "userId": "user123",
  "loginFrequency": 15,
  "failedLoginAttempts": 2,
  "dataAccessPatterns": [0.8, 0.6, 0.9],
  "workingHoursAnomaly": 0.3,
  "taskCompletionRate": 0.85
}

// Response
{
  "riskLevel": "medium",
  "riskScore": 0.62,
  "factors": ["failedLoginAttempts", "workingHoursAnomaly"],
  "recommendations": ["Enable 2FA", "Review access patterns"]
}
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "features": {
    "loginTime": "2026-04-20T03:00:00Z",
    "ipAddress": "192.168.1.100",
    "accessedResources": ["admin-panel", "user-data"],
    "dataVolume": 5000
  }
}

// Response
{
  "isAnomaly": true,
  "anomalyScore": 0.87,
  "anomalyType": "unusual-access-time",
  "details": "Login detected outside normal working hours"
}
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
POST /api/ml/burnout-analysis
{
  "userId": "user123",
  "workload": {
    "tasksAssigned": 25,
    "tasksCompleted": 18,
    "averageWorkHours": 52,
    "weekendWork": 6,
    "overtimeHours": 20
  },
  "period": "last-30-days"
}

// Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.78,
  "factors": ["excessive-overtime", "weekend-work", "high-task-load"],
  "recommendations": [
    "Reduce task assignment by 30%",
    "Schedule mandatory time off",
    "Redistribute workload"
  ]
}
```

### Ticket Classification

```javascript
// Auto-classify and route support ticket
POST /api/ml/classify-ticket
{
  "ticketId": "ticket123",
  "title": "Database connection timeout",
  "description": "Application throws timeout error when connecting to MongoDB",
  "priority": "high"
}

// Response
{
  "category": "technical",
  "subCategory": "database",
  "suggestedAssignee": "db-admin-team",
  "estimatedResolutionTime": "4 hours",
  "confidence": 0.92
}
```

### Predictive Project Insights

```javascript
// Predict project completion
POST /api/ml/project-insights
{
  "projectId": "proj123",
  "tasks": [
    {"id": "task1", "status": "done", "estimatedHours": 8, "actualHours": 10},
    {"id": "task2", "status": "in-progress", "estimatedHours": 16, "actualHours": 12},
    {"id": "task3", "status": "todo", "estimatedHours": 24, "actualHours": 0}
  ],
  "deadline": "2026-05-15"
}

// Response
{
  "completionProbability": 0.65,
  "predictedDelay": "5 days",
  "riskyTasks": ["task3"],
  "recommendations": [
    "Add 1 additional developer",
    "Reduce scope of task3",
    "Extend deadline by 1 week"
  ]
}
```

## Frontend Integration Patterns

### Authentication Hook

```javascript
// src/hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

export const useAuth = () => {
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
      const response = await axios.get(`${API_URL}/api/auth/me`);
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
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

  return { user, loading, login, logout };
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks/user/${userId}`);
      const tasksByStatus = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(tasksByStatus);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.put(`${API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = (e, status) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    moveTask(taskId, status);
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status === 'inProgress' ? 'In Progress' : status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task.id}
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task.id)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>
                {task.priority}
              </span>
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
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalyticsDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutAlerts: [],
    anomalies: [],
    projectInsights: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch users with risk assessment
      const usersResponse = await axios.get(`${API_URL}/api/users`);
      
      const riskAssessments = await Promise.all(
        usersResponse.data.map(async (user) => {
          const riskResponse = await axios.post(`${ML_API_URL}/api/ml/predict-risk`, {
            userId: user.id,
            loginFrequency: user.loginFrequency || 10,
            failedLoginAttempts: user.failedLoginAttempts || 0,
            dataAccessPatterns: user.dataAccessPatterns || [0.5, 0.5, 0.5],
            workingHoursAnomaly: user.workingHoursAnomaly || 0.2,
            taskCompletionRate: user.taskCompletionRate || 0.8
          });
          return { user, risk: riskResponse.data };
        })
      );

      const highRiskUsers = riskAssessments.filter(
        r => r.risk.riskLevel === 'high' || r.risk.riskLevel === 'medium'
      );

      setAnalytics(prev => ({ ...prev, riskUsers: highRiskUsers }));
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  const analyzeBurnout = async (userId) => {
    try {
      const response = await axios.post(`${ML_API_URL}/api/ml/burnout-analysis`, {
        userId,
        workload: {
          tasksAssigned: 25,
          tasksCompleted: 18,
          averageWorkHours: 45,
          weekendWork: 2,
          overtimeHours: 10
        },
        period: 'last-30-days'
      });
      return response.data;
    } catch (error) {
      console.error('Burnout analysis failed:', error);
    }
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <section className="risk-alerts">
        <h3>Risk Alerts</h3>
        {analytics.riskUsers.map(({ user, risk }) => (
          <div key={user.id} className={`alert-card ${risk.riskLevel}`}>
            <h4>{user.name}</h4>
            <p>Risk Level: {risk.riskLevel}</p>
            <p>Score: {(risk.riskScore * 100).toFixed(0)}%</p>
            <ul>
              {risk.recommendations?.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Database Schema Examples

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide name'],
    trim: true
  },
  email: {
    type: String,
    required: [true, 'Please provide email'],
    unique: true,
    lowercase: true,
    match: [/^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/, 'Please provide valid email']
  },
  password: {
    type: String,
    required: [true, 'Please provide password'],
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'manager', 'admin'],
    default: 'user'
  },
  department: String,
  loginFrequency: { type: Number, default: 0 },
  failedLoginAttempts: { type: Number, default: 0 },
  taskCompletionRate: { type: Number, default: 0 },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

// Hash password before saving
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) {
    next();
  }
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Sign JWT token
UserSchema.methods.getSignedJwtToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE
  });
};

// Compare passwords
UserSchema.methods.matchPassword = async function(enteredPassword) {
  return await bcrypt.compare(enteredPassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const TaskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, 'Please provide task title'],
    trim: true
  },
  description: {
    type: String,
    required: [true, 'Please provide task description']
  },
  assignedTo: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.ObjectId,
    ref: 'User',
    required: true
  },
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
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  estimatedHours: Number,
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

module.exports = mongoose.model('Task', TaskSchema);
```

## ML Model Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import numpy as np
import pickle
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        self.scaler = StandardScaler()
        self.model = None
        self.load_model()
    
    def load_model(self):
        if os.path.exists(self.model_path):
            with open(self.model_path, 'rb') as f:
                saved_data = pickle.load(f)
                self.model = saved_data['model']
                self.scaler = saved_data['scaler']
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
    
    def prepare_features(self, data):
        features = np.array([[
            data.get('loginFrequency', 0),
            data.get('failedLoginAttempts', 0),
            np.mean(data.get('dataAccessPatterns', [0.5])),
            data.get('workingHoursAnomaly', 0),
            data.get('taskCompletionRate', 0.5)
        ]])
        return features
    
    def predict(self, user_data):
        features = self.prepare_features(user_data)
        
        if self.model.tree_ is None:
            # Model not trained, return default prediction
            return self._default_prediction(user_data)
        
        features_scaled = self.scaler.transform(features)
        risk_proba = self.model.predict_proba(features_scaled)[0]
        risk_level = self._get_risk_level(risk_proba)
        
        return {
            'riskLevel': risk_level,
            'riskScore': float(risk_proba[1]),
            'factors': self._identify_risk_factors(user_data),
            'recommendations': self._generate_recommendations(risk_level, user_data)
        }
    
    def _get_risk_level(self, proba):
        risk_score = proba[1]
        if risk_score > 0.7:
            return 'high'
        elif risk_score > 0.4:
            return 'medium'
        return 'low'
    
    def _identify_risk_factors(self, data):
        factors = []
        if data.get('failedLoginAttempts', 0) > 3:
            factors.append('failedLoginAttempts')
        if data.get('workingHoursAnomaly', 0) > 0.5:
            factors.append('workingHoursAnomaly')
        if data.get('taskCompletionRate', 1.0) < 0.5:
            factors.append('lowTaskCompletion')
        return factors
    
    def _generate_recommendations(self, risk_level, data):
        recommendations = []
        if risk_level in ['high', 'medium']:
            if data.get('failedLoginAttempts', 0) > 3:
                recommendations.append('Enable 2FA')
                recommendations.append('Review access permissions')
            if data.get('workingHoursAnomaly', 0) > 0.5:
                recommendations.append('Review access patterns')
        return recommendations
    
    def _default_prediction(self, data):
        score = 0.3
        if data.get('failedLoginAttempts', 0) > 3:
            score += 0.3
        if data.get('workingHoursAnomaly', 0) > 0.5:
            score += 0.2
        
        return {
            'riskLevel': self._get_risk_level([1-score, score]),
            'riskScore': score,
            'factors': self._identify_risk_factors(data),
            'recommendations': self._generate_recommendations(
                self._get_risk_level([1-score, score]), data
            )
        }
```

### FastAPI ML Service Main

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Dict, Optional
from models.risk_predictor import RiskPredictor
from models.anomaly_detector import AnomalyDetector
from models.burnout_analyzer import BurnoutAnalyzer
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

# Initialize models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

# Request models
class RiskPredictionRequest(BaseModel):
    userId: str
    loginFrequency: int
    failedLoginAttempts: int
    dataAccessPatterns: List[float]
    workingHoursAnomaly: float
    taskCompletionRate: float

class AnomalyDetectionRequest(BaseModel):
    userId: str
    features: Dict

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workload: Dict
    period: str

class TicketClassificationRequest(BaseModel):
    ticketId: str
    title: str
    description: str
    priority: str

# Endpoints
@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutAnalysisRequest):
    try:
        result = burnout_analyzer.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        # Simple keyword-based classification
        title_desc = f"{request.title} {request.description}".lower()
        
        category = "general"
        sub_category = "other"
        
        if any(word in title_desc for word in ['database', 'db', 'mongo', 'sql']):
            category = "technical"
            sub_category = "database"
        elif any(word in title_desc for word in ['login', 'password', 'access', 'permission']):
            category = "security"
            sub_category = "authentication"
        elif any(word in title_desc for word in ['bug', 'error', 'crash', 'issue']):
            category = "technical"
            sub_category = "bug"
        
        return {
            "category": category,
            "subCategory": sub_category,
            "suggestedAssignee": f"{sub_category}-team",
            "estimatedResolutionTime": "4 hours",
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}
```

## Common Configuration Patterns

### Environment Variables

```bash
# Backend .env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000

# ML Service .env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
RISK_THRESHOLD_HIGH=0.7
RISK_THRESHOLD_MEDIUM=0.4

# Frontend .env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

### Axios Instance Configuration

```javascript
// src/utils/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor
api.interceptors.request.use(
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
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Troubleshooting

### JWT Authentication Issues

```javascript
// Check if token is valid
const verifyToken = async () => {
  try {
    const response = await axios.get(`${API_URL}/api/auth/me`);
    console.log('Token valid, user:', response.data);
  } catch (error) {
    console.error('Token invalid:', error.response?.status);
    // Clear invalid token
    localStorage.removeItem('token');
  }
};

// Handle token expiration
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('token');
      window.location
