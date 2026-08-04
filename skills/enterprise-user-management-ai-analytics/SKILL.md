---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management dashboard with AI insights"
  - "implement AI-powered ticket classification system"
  - "build task management with burnout detection"
  - "add risk prediction to user management"
  - "integrate AI analytics into enterprise system"
  - "develop user management with ML service"
  - "create admin dashboard with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with machine learning capabilities. It provides:

- **User Management**: Role-based access control, authentication, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment
- **Support System**: AI-classified ticket routing and management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization analytics, audit logs, alerts

The system consists of three main components:
- **Frontend**: React.js application
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI service with scikit-learn and River for online learning

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
JWT_SECRET=\${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Key APIs and Endpoints

### Backend API Endpoints

#### Authentication

```javascript
// Register new user
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
// Returns: { token: "jwt-token", user: {...} }
```

#### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer ${JWT_TOKEN}

// Update user
PUT /api/users/:id
Authorization: Bearer ${JWT_TOKEN}
{
  "name": "Updated Name",
  "role": "admin",
  "isActive": true
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer ${JWT_TOKEN}
```

#### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "userId",
  "status": "todo", // todo, in-progress, done
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer ${JWT_TOKEN}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
}
```

#### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Authorization: Bearer ${JWT_TOKEN}
{
  "title": "Login issue",
  "description": "Cannot log in with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get tickets (with AI classification)
GET /api/tickets
Authorization: Bearer ${JWT_TOKEN}

// Update ticket
PATCH /api/tickets/:id
{
  "status": "resolved",
  "assignedTo": "adminId"
}
```

### ML Service Endpoints

#### Risk Prediction

```javascript
// Predict user risk score
POST /api/ml/predict-risk
Content-Type: application/json
{
  "userId": "user123",
  "features": {
    "taskCompletionRate": 0.75,
    "averageDelay": 2.5,
    "activeTickets": 3,
    "hoursWorked": 45,
    "lastLoginDays": 1
  }
}
// Returns: { riskScore: 0.35, riskLevel: "medium" }
```

#### Anomaly Detection

```javascript
// Detect anomalies in user behavior
POST /api/ml/detect-anomaly
{
  "userId": "user123",
  "behaviorData": {
    "loginTime": "03:00",
    "loginLocation": "unusual-ip",
    "activityPattern": [0, 0, 1, 5, 20, 15]
  }
}
// Returns: { isAnomaly: true, confidence: 0.87 }
```

#### Burnout Detection

```javascript
// Analyze burnout risk
POST /api/ml/detect-burnout
{
  "userId": "user123",
  "workloadData": {
    "weeklyHours": [50, 55, 60, 58, 62],
    "tasksCompleted": [5, 6, 4, 3, 2],
    "ticketsRaised": [0, 1, 2, 3, 4],
    "overtimeHours": 15
  }
}
// Returns: { burnoutScore: 0.72, recommendation: "reduce workload" }
```

#### Ticket Classification

```javascript
// Auto-classify support ticket
POST /api/ml/classify-ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when trying to access admin dashboard",
  "metadata": {
    "userRole": "user",
    "previousTickets": 2
  }
}
// Returns: { category: "access-control", priority: "high", suggestedAssignee: "admin-team" }
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (err) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    setToken(res.data.token);
    setUser(res.data.user);
    localStorage.setItem('token', res.data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Management Component

```javascript
// src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    'in-progress': [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks`);
    const grouped = res.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], 'in-progress': [], done: [] });
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus }
    );
    fetchTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select
                value={task.status}
                onChange={(e) => updateTaskStatus(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      setLoading(true);
      
      // Fetch multiple AI insights
      const [risk, burnout, anomalies] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`, {
          userId,
          features: await getUserFeatures(userId)
        }),
        axios.post(`${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`, {
          userId,
          workloadData: await getWorkloadData(userId)
        }),
        axios.post(`${process.env.REACT_APP_ML_URL}/api/ml/detect-anomaly`, {
          userId,
          behaviorData: await getBehaviorData(userId)
        })
      ]);

      setAnalytics({
        risk: risk.data,
        burnout: burnout.data,
        anomalies: anomalies.data
      });
    } catch (err) {
      console.error('Failed to fetch analytics:', err);
    } finally {
      setLoading(false);
    }
  };

  const getUserFeatures = async (userId) => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/users/${userId}/features`);
    return res.data;
  };

  const getWorkloadData = async (userId) => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/users/${userId}/workload`);
    return res.data;
  };

  const getBehaviorData = async (userId) => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/api/users/${userId}/behavior`);
    return res.data;
  };

  if (loading) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="risk-indicator">
        <h3>Risk Score</h3>
        <div className={`score ${analytics.risk.riskLevel}`}>
          {(analytics.risk.riskScore * 100).toFixed(0)}%
        </div>
        <p>Level: {analytics.risk.riskLevel}</p>
      </div>

      <div className="burnout-indicator">
        <h3>Burnout Risk</h3>
        <div className="score">
          {(analytics.burnout.burnoutScore * 100).toFixed(0)}%
        </div>
        <p>{analytics.burnout.recommendation}</p>
      </div>

      <div className="anomaly-alerts">
        <h3>Security Alerts</h3>
        {analytics.anomalies.isAnomaly ? (
          <div className="alert warning">
            Unusual activity detected (confidence: {(analytics.anomalies.confidence * 100).toFixed(0)}%)
          </div>
        ) : (
          <div className="alert success">No anomalies detected</div>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Backend Implementation Patterns

### User Model (MongoDB)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    select: false
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
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
UserSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
UserSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.protect = async (req, res, next) => {
  let token;

  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }

  if (!token) {
    return res.status(401).json({ error: 'Not authorized to access this route' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id);
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Not authorized to access this route' });
  }
};

exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'User role not authorized' });
    }
    next();
  };
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    res.status(201).json(task);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.params.userId })
      .populate('assignedTo', 'name email')
      .populate('createdBy', 'name');
    res.json(tasks);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { status: req.body.status },
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }
    
    res.json(task);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
};
```

## ML Service Implementation

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import numpy as np
from typing import List, Dict
import joblib
from pathlib import Path

app = FastAPI(title="Enterprise User Management ML Service")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Load models
MODEL_PATH = Path("./models")
risk_model = joblib.load(MODEL_PATH / "risk_predictor.pkl") if (MODEL_PATH / "risk_predictor.pkl").exists() else None

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict[str, float]

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workloadData: Dict

class AnomalyDetectionRequest(BaseModel):
    userId: str
    behaviorData: Dict

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    metadata: Dict

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    """Predict user risk score based on behavior features"""
    try:
        features = request.features
        
        # Extract feature vector
        feature_vector = np.array([
            features.get("taskCompletionRate", 0.5),
            features.get("averageDelay", 0),
            features.get("activeTickets", 0),
            features.get("hoursWorked", 40),
            features.get("lastLoginDays", 0)
        ]).reshape(1, -1)
        
        # Predict risk score
        if risk_model:
            risk_score = float(risk_model.predict_proba(feature_vector)[0][1])
        else:
            # Fallback calculation
            risk_score = calculate_risk_score(features)
        
        risk_level = "low" if risk_score < 0.3 else "medium" if risk_score < 0.7 else "high"
        
        return {
            "riskScore": risk_score,
            "riskLevel": risk_level,
            "factors": analyze_risk_factors(features)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    """Detect burnout risk based on workload patterns"""
    try:
        workload = request.workloadData
        
        weekly_hours = workload.get("weeklyHours", [40] * 5)
        tasks_completed = workload.get("tasksCompleted", [5] * 5)
        tickets_raised = workload.get("ticketsRaised", [0] * 5)
        overtime = workload.get("overtimeHours", 0)
        
        # Calculate burnout indicators
        avg_hours = np.mean(weekly_hours)
        hour_trend = np.polyfit(range(len(weekly_hours)), weekly_hours, 1)[0]
        productivity_trend = np.polyfit(range(len(tasks_completed)), tasks_completed, 1)[0]
        
        # Burnout score calculation
        burnout_score = 0.0
        
        if avg_hours > 50:
            burnout_score += 0.3
        if hour_trend > 2:  # Increasing hours
            burnout_score += 0.2
        if productivity_trend < -0.5:  # Decreasing productivity
            burnout_score += 0.3
        if overtime > 10:
            burnout_score += 0.2
        
        burnout_score = min(burnout_score, 1.0)
        
        # Generate recommendation
        if burnout_score > 0.7:
            recommendation = "Critical: Immediate workload reduction needed"
        elif burnout_score > 0.5:
            recommendation = "Warning: Consider reducing workload and taking breaks"
        else:
            recommendation = "Workload appears manageable"
        
        return {
            "burnoutScore": burnout_score,
            "recommendation": recommendation,
            "metrics": {
                "averageHours": avg_hours,
                "hoursTrend": "increasing" if hour_trend > 0 else "decreasing",
                "productivityTrend": "decreasing" if productivity_trend < 0 else "increasing"
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyDetectionRequest):
    """Detect anomalies in user behavior"""
    try:
        behavior = request.behaviorData
        
        # Extract behavior features
        login_time = behavior.get("loginTime", "09:00")
        login_location = behavior.get("loginLocation", "normal")
        activity_pattern = behavior.get("activityPattern", [])
        
        # Anomaly detection logic
        anomalies = []
        confidence = 0.0
        
        # Check login time (unusual if outside 6 AM - 10 PM)
        hour = int(login_time.split(":")[0])
        if hour < 6 or hour > 22:
            anomalies.append("Unusual login time")
            confidence += 0.3
        
        # Check location
        if login_location == "unusual-ip":
            anomalies.append("Login from unusual location")
            confidence += 0.4
        
        # Check activity pattern
        if activity_pattern:
            avg_activity = np.mean(activity_pattern)
            if avg_activity > 50:  # Unusually high activity
                anomalies.append("Unusually high activity")
                confidence += 0.3
        
        is_anomaly = confidence > 0.5
        
        return {
            "isAnomaly": is_anomaly,
            "confidence": min(confidence, 1.0),
            "anomalies": anomalies
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    """Automatically classify support tickets"""
    try:
        title = request.title.lower()
        description = request.description.lower()
        combined_text = f"{title} {description}"
        
        # Simple keyword-based classification
        category = "general"
        priority = "medium"
        
        # Category detection
        if any(word in combined_text for word in ["login", "password", "access", "403", "401"]):
            category = "access-control"
            priority = "high"
        elif any(word in combined_text for word in ["error", "crash", "bug", "broken"]):
            category = "technical"
            priority = "high"
        elif any(word in combined_text for word in ["slow", "performance", "lag"]):
            category = "performance"
            priority = "medium"
        elif any(word in combined_text for word in ["feature", "request", "add"]):
            category = "feature-request"
            priority = "low"
        
        # Suggested assignee based on category
        assignee_map = {
            "access-control": "security-team",
            "technical": "dev-team",
            "performance": "ops-team",
            "feature-request": "product-team",
            "general": "support-team"
        }
        
        return {
            "category": category,
            "priority": priority,
            "suggestedAssignee": assignee_map.get(category, "support-team"),
            "confidence": 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

def calculate_risk_score(features: Dict[str, float]) -> float:
    """Calculate risk score from features"""
    score = 0.0
    
    completion_rate = features.get("taskCompletionRate", 1.0)
    if completion_rate < 0.5:
        score += 0.3
    
    avg_delay = features.get("averageDelay", 0)
    if avg_delay > 3:
        score += 0.3
    
    active_tickets = features.get("activeTickets", 0)
    if active_tickets > 5:
        score += 0.2
    
    last_login = features.get("lastLoginDays", 0)
    if last_login > 7:
        score += 0.2
    
    return min(score, 1.0)

def analyze_risk_factors(features: Dict[str, float]) -> List[str]:
    """Analyze which factors contribute to risk"""
    factors = []
    
    if features.get("taskCompletionRate", 1.0) < 0.5:
        factors.append("Low task completion rate")
    if features.get("averageDelay", 0) > 3:
        factors.append("High average delay")
    if features.get("activeTickets", 0) > 5:
        factors.append("Many active support tickets")
    if features.get("lastLoginDays", 0) > 7:
        factors.append("Inactive for extended period")
    
    return factors if factors else ["No significant risk factors"]

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Analytics"}
```

## Configuration

### Backend Environment Variables

```bash
# backend/.env
PORT=5000
NODE_ENV=development

# MongoDB
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
# Or use MongoDB Atlas
# MONGODB_URI=mongodb+srv://${DB_USER}:${DB_PASSWORD}@cluster.mongodb.net/enterprise

# JWT
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional for notifications)
SMTP_HOST=${SMTP_HOST}
SMTP_PORT=${SMTP_PORT}
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}
```

### Frontend Environment Variables

```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_NAME="Enterprise User Management"
```

### ML Service Environment Variables

```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=info
CACHE_ENABLED=true
```

## Common Patterns

### Protected Routes in Frontend

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### API Service Layer

```javascript
// src/services/api.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;
const ML_URL = process.env.REACT_APP_ML_URL;

// Set auth token
export const setAuthToken = (token) => {
  if (token) {
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  } else {
    delete axios.defaults.headers.common['Authorization'];
  }
};

// User API
export const userAPI = {
  getAll: () => axios.get(`${API_URL}/api/users`),
  getById: (id) => axios.get(`${API_URL}/api/users/${id}`),
  update: (id, data) => axios.put(`${API_URL}/api/users/${id}`, data),
  delete: (id) => axios.delete(`${API_URL}/api/users/${id}`)
};

// Task API
export const taskAPI = {
  create: (data) => axios.post(`${API_URL}/api/tasks`, data),
  getByUser: (userId) => axios.get(`${API_URL}/api/tasks/user/${userId}`),
  updateStatus: (id, status) => axios.patch(`${API_URL}/api/tasks/${id}/status`, { status })
};

// Ticket API
export const ticketAPI = {
  create: (data) => axios.post(`${API_URL}/api/tickets`, data),
  getAll: () => axios.get(`${API_URL}/api/tickets`),
  update: (id, data) => axios.patch(`${API_URL}/api/tickets/${id}`, data)
};

// ML API
export const mlAPI = {
  predictRisk: (userId, features) => 
    axios.post(`${ML_URL}/api/ml/predict-risk`, { userId, features }),
  detectBurnout: (userId, workloadData) => 
    axios.post(`${ML_URL}/api/ml/detect-burnout`, { userId, workloadData }),
  detectAnomaly: (userId, behaviorData
