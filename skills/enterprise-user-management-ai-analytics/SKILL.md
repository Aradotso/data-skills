---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with risk detection"
  - "build admin dashboard with AI insights"
  - "add AI ticket classification"
  - "implement burnout detection system"
  - "configure user management with ML service"
  - "integrate AI analytics for user behavior"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines traditional CRUD operations with AI-powered analytics. The system provides risk detection, anomaly detection, burnout analysis, predictive project insights, and intelligent ticket classification. Built with React frontend, Node.js backend, MongoDB database, and FastAPI ML service using scikit-learn and River for online learning.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or cloud)

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_key
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

Create `.env` file in ml-service directory:

```env
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

Create `.env` file in frontend directory:

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

1. **Frontend (React)**: User interface for admin and user dashboards
2. **Backend (Node.js)**: REST API for user management, tasks, and tickets
3. **ML Service (FastAPI)**: AI models for analytics and predictions

## Backend API Reference

### Authentication

```javascript
// Login
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Response
{
  "token": "jwt_token_here",
  "user": {
    "id": "user_id",
    "name": "User Name",
    "role": "admin" // or "user"
  }
}
```

### User Management (Admin Only)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <token>" }

// Create user
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:id
{
  "name": "John Doe Updated",
  "role": "admin"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <token>" }

// Create task
POST /api/tasks
{
  "title": "Implement feature X",
  "description": "Details here",
  "assignedTo": "user_id",
  "priority": "high", // low, medium, high
  "dueDate": "2026-05-01",
  "status": "todo" // todo, inprogress, done
}

// Update task status
PUT /api/tasks/:id
{
  "status": "inprogress"
}

// Track time
POST /api/tasks/:id/time
{
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Login issue",
  "description": "Cannot login to system",
  "priority": "medium"
}

// Get user tickets
GET /api/tickets

// Get all tickets (Admin)
GET /api/tickets/all

// Update ticket
PUT /api/tickets/:id
{
  "status": "resolved",
  "resolution": "Password reset performed"
}
```

## ML Service API Reference

### Risk Prediction

```javascript
// Predict user risk score
POST http://localhost:8000/api/ml/predict-risk
{
  "userId": "user_id",
  "taskCompletionRate": 0.75,
  "avgTaskDuration": 7200,
  "ticketCount": 5,
  "loginFrequency": 20
}

// Response
{
  "riskScore": 0.35,
  "riskLevel": "medium", // low, medium, high
  "factors": ["low completion rate", "high ticket count"]
}
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
POST http://localhost:8000/api/ml/detect-anomaly
{
  "userId": "user_id",
  "loginTime": "2026-04-15T02:30:00Z",
  "ipAddress": "192.168.1.100",
  "action": "bulk_delete",
  "resourceAccess": ["sensitive_data"]
}

// Response
{
  "isAnomaly": true,
  "anomalyScore": 0.85,
  "reasons": ["unusual login time", "suspicious action pattern"]
}
```

### Burnout Detection

```javascript
// Analyze user burnout risk
POST http://localhost:8000/api/ml/burnout-analysis
{
  "userId": "user_id",
  "weeklyHours": 65,
  "taskLoad": 25,
  "overtimeHours": 15,
  "missedDeadlines": 3
}

// Response
{
  "burnoutRisk": "high",
  "score": 0.78,
  "recommendations": [
    "Reduce task load by 30%",
    "Schedule mandatory time off"
  ]
}
```

### Ticket Classification

```javascript
// Classify support ticket
POST http://localhost:8000/api/ml/classify-ticket
{
  "title": "Cannot access database",
  "description": "Getting connection timeout errors when trying to access production DB"
}

// Response
{
  "category": "technical",
  "subCategory": "database",
  "priority": "high",
  "suggestedAssignee": "devops_team",
  "confidence": 0.92
}
```

### Predictive Project Insights

```javascript
// Predict project delays
POST http://localhost:8000/api/ml/predict-delay
{
  "projectId": "project_id",
  "plannedDuration": 30,
  "currentProgress": 0.40,
  "daysElapsed": 20,
  "teamSize": 5,
  "blockerCount": 3
}

// Response
{
  "delayProbability": 0.68,
  "estimatedDelay": 10, // days
  "completionDate": "2026-05-20",
  "suggestions": [
    "Add 2 more team members",
    "Resolve current blockers"
  ]
}
```

## Frontend Integration Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

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
      const res = await axios.get(`${API_URL}/api/auth/me`);
      setUser(res.data);
    } catch (error) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${API_URL}/api/auth/login`, {
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
    <AuthContext.Provider value={{ user, login, logout, isAdmin: user?.role === 'admin' }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Fetching AI Analytics

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_SERVICE_URL = process.env.REACT_APP_ML_SERVICE_URL;

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [riskData, setRiskData] = useState([]);

  useEffect(() => {
    fetchUsersWithRisk();
  }, []);

  const fetchUsersWithRisk = async () => {
    try {
      const usersRes = await axios.get('/api/users');
      const usersData = usersRes.data;

      // Get risk scores for each user
      const riskPromises = usersData.map(async (user) => {
        const metricsRes = await axios.get(`/api/analytics/user/${user._id}`);
        const metrics = metricsRes.data;

        const riskRes = await axios.post(`${ML_SERVICE_URL}/api/ml/predict-risk`, {
          userId: user._id,
          taskCompletionRate: metrics.completionRate,
          avgTaskDuration: metrics.avgDuration,
          ticketCount: metrics.ticketCount,
          loginFrequency: metrics.loginFrequency
        });

        return {
          ...user,
          riskScore: riskRes.data.riskScore,
          riskLevel: riskRes.data.riskLevel
        };
      });

      const enrichedUsers = await Promise.all(riskPromises);
      setUsers(enrichedUsers);
    } catch (error) {
      console.error('Error fetching user risk data:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>User Risk Overview</h1>
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Department</th>
            <th>Risk Level</th>
            <th>Risk Score</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user._id} className={`risk-${user.riskLevel}`}>
              <td>{user.name}</td>
              <td>{user.department}</td>
              <td>{user.riskLevel}</td>
              <td>{(user.riskScore * 100).toFixed(1)}%</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default AdminDashboard;
```

### Task Kanban Board

```javascript
// src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inprogress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const res = await axios.get('/api/tasks');
      const grouped = res.data.reduce((acc, task) => {
        acc[task.status] = acc[task.status] || [];
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`/api/tasks/${taskId}`, { status: newStatus });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderTask = (task) => (
    <div key={task._id} className="task-card">
      <h4>{task.title}</h4>
      <p>{task.description}</p>
      <span className={`priority-${task.priority}`}>{task.priority}</span>
      <select 
        value={task.status} 
        onChange={(e) => updateTaskStatus(task._id, e.target.value)}
      >
        <option value="todo">To Do</option>
        <option value="inprogress">In Progress</option>
        <option value="done">Done</option>
      </select>
    </div>
  );

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do ({tasks.todo.length})</h3>
        {tasks.todo.map(renderTask)}
      </div>
      <div className="column">
        <h3>In Progress ({tasks.inprogress.length})</h3>
        {tasks.inprogress.map(renderTask)}
      </div>
      <div className="column">
        <h3>Done ({tasks.done.length})</h3>
        {tasks.done.map(renderTask)}
      </div>
    </div>
  );
};

export default TaskBoard;
```

## Backend Implementation Patterns

### User Controller with JWT

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

const JWT_SECRET = process.env.JWT_SECRET;

exports.register = async (req, res) => {
  try {
    const { name, email, password, role, department } = req.body;

    // Check if user exists
    let user = await User.findOne({ email });
    if (user) {
      return res.status(400).json({ message: 'User already exists' });
    }

    // Hash password
    const salt = await bcrypt.genSalt(10);
    const hashedPassword = await bcrypt.hash(password, salt);

    // Create user
    user = new User({
      name,
      email,
      password: hashedPassword,
      role: role || 'user',
      department
    });

    await user.save();

    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.status(201).json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Check user
    const user = await User.findOne({ email });
    if (!user) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }

    // Verify password
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) {
      return res.status(400).json({ message: 'Invalid credentials' });
    }

    // Update last login
    user.lastLogin = new Date();
    await user.save();

    // Generate token
    const token = jwt.sign(
      { userId: user._id, role: user.role },
      JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.json({
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET;

exports.authenticate = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ message: 'No token, authorization denied' });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Token is not valid' });
  }
};

exports.requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');
const axios = require('axios');

const ML_SERVICE_URL = process.env.ML_SERVICE_URL;

exports.createTask = async (req, res) => {
  try {
    const { title, description, assignedTo, priority, dueDate } = req.body;

    const task = new Task({
      title,
      description,
      assignedTo,
      createdBy: req.user.userId,
      priority,
      dueDate,
      status: 'todo'
    });

    await task.save();

    // Predict potential delay
    try {
      const delayRes = await axios.post(`${ML_SERVICE_URL}/api/ml/predict-delay`, {
        projectId: task._id,
        plannedDuration: Math.ceil((new Date(dueDate) - new Date()) / (1000 * 60 * 60 * 24)),
        currentProgress: 0,
        daysElapsed: 0,
        teamSize: 1,
        blockerCount: 0
      });

      task.delayPrediction = delayRes.data;
      await task.save();
    } catch (mlError) {
      console.error('ML prediction failed:', mlError);
    }

    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({
      $or: [
        { assignedTo: req.user.userId },
        { createdBy: req.user.userId }
      ]
    }).populate('assignedTo', 'name email').sort({ createdAt: -1 });

    res.json(tasks);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const { status, progress } = req.body;
    const task = await Task.findById(req.params.id);

    if (!task) {
      return res.status(404).json({ message: 'Task not found' });
    }

    if (status) task.status = status;
    if (progress !== undefined) task.progress = progress;

    if (status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100, random_state=42)
            self._train_initial_model()
    
    def _train_initial_model(self):
        # Initial training with synthetic data
        # In production, replace with historical data
        X = np.random.rand(1000, 4)
        y = (X[:, 0] < 0.5).astype(int)  # Simple rule for demo
        self.model.fit(X, y)
        joblib.dump(self.model, self.model_path)
    
    def predict(self, features):
        """
        Predict risk level
        features: [taskCompletionRate, avgTaskDuration, ticketCount, loginFrequency]
        """
        features_array = np.array(features).reshape(1, -1)
        risk_score = self.model.predict_proba(features_array)[0][1]
        
        if risk_score < 0.3:
            risk_level = 'low'
        elif risk_score < 0.7:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': self._identify_factors(features, risk_score)
        }
    
    def _identify_factors(self, features, risk_score):
        factors = []
        if features[0] < 0.6:
            factors.append('low completion rate')
        if features[2] > 10:
            factors.append('high ticket count')
        if features[3] < 5:
            factors.append('low engagement')
        return factors
```

### Anomaly Detection

```python
# ml-service/models/anomaly_detector.py
from river import anomaly
from river import preprocessing
import json
import os

class AnomalyDetector:
    def __init__(self):
        self.model = anomaly.HalfSpaceTrees(n_trees=10, height=8, window_size=250)
        self.scaler = preprocessing.StandardScaler()
        self.history_file = './models/anomaly_history.json'
        self.load_history()
    
    def load_history(self):
        if os.path.exists(self.history_file):
            with open(self.history_file, 'r') as f:
                history = json.load(f)
                for record in history:
                    self.learn_one(record['features'])
    
    def detect(self, features_dict):
        """
        Detect anomalies in user behavior
        features_dict: {loginHour, actionCount, ipHash, resourceSensitivity}
        """
        features = {
            'login_hour': features_dict.get('loginHour', 12),
            'action_count': features_dict.get('actionCount', 5),
            'ip_hash': features_dict.get('ipHash', 0),
            'resource_sensitivity': features_dict.get('resourceSensitivity', 0)
        }
        
        # Scale features
        scaled = self.scaler.learn_one(features).transform_one(features)
        
        # Get anomaly score
        score = self.model.score_one(scaled)
        is_anomaly = score > 0.7
        
        # Update model
        self.model.learn_one(scaled)
        
        reasons = []
        if features['login_hour'] < 6 or features['login_hour'] > 22:
            reasons.append('unusual login time')
        if features['action_count'] > 50:
            reasons.append('high activity volume')
        if features['resource_sensitivity'] > 0.8:
            reasons.append('accessing sensitive resources')
        
        return {
            'isAnomaly': is_anomaly,
            'anomalyScore': float(score),
            'reasons': reasons if is_anomaly else []
        }
    
    def learn_one(self, features):
        scaled = self.scaler.learn_one(features).transform_one(features)
        self.model.learn_one(scaled)
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.anomaly_detector import AnomalyDetector
import uvicorn

app = FastAPI(title="Enterprise UMS ML Service")

risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()

class RiskRequest(BaseModel):
    userId: str
    taskCompletionRate: float
    avgTaskDuration: float
    ticketCount: int
    loginFrequency: int

class AnomalyRequest(BaseModel):
    userId: str
    loginTime: str
    ipAddress: str
    action: str
    resourceAccess: list

class BurnoutRequest(BaseModel):
    userId: str
    weeklyHours: float
    taskLoad: int
    overtimeHours: float
    missedDeadlines: int

class TicketRequest(BaseModel):
    title: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskRequest):
    try:
        features = [
            request.taskCompletionRate,
            request.avgTaskDuration / 3600,  # Convert to hours
            request.ticketCount,
            request.loginFrequency
        ]
        result = risk_predictor.predict(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-anomaly")
async def detect_anomaly(request: AnomalyRequest):
    try:
        from datetime import datetime
        login_dt = datetime.fromisoformat(request.loginTime.replace('Z', '+00:00'))
        
        features = {
            'loginHour': login_dt.hour,
            'actionCount': 1,
            'ipHash': hash(request.ipAddress) % 1000,
            'resourceSensitivity': 0.8 if 'sensitive' in str(request.resourceAccess) else 0.2
        }
        
        result = anomaly_detector.detect(features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-analysis")
async def analyze_burnout(request: BurnoutRequest):
    try:
        # Simple burnout scoring
        score = 0
        recommendations = []
        
        if request.weeklyHours > 50:
            score += 0.3
            recommendations.append("Reduce working hours to under 45/week")
        
        if request.taskLoad > 20:
            score += 0.25
            recommendations.append("Reduce task load by 30%")
        
        if request.overtimeHours > 10:
            score += 0.25
            recommendations.append("Limit overtime hours")
        
        if request.missedDeadlines > 2:
            score += 0.2
            recommendations.append("Review deadline feasibility")
        
        if score < 0.3:
            risk = "low"
        elif score < 0.6:
            risk = "medium"
        else:
            risk = "high"
            recommendations.append("Schedule mandatory time off")
        
        return {
            "burnoutRisk": risk,
            "score": score,
            "recommendations": recommendations
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        text = f"{request.title} {request.description}".lower()
        
        # Simple keyword-based classification
        categories = {
            'technical': ['error', 'bug', 'crash', 'database', 'api', 'server'],
            'access': ['login', 'password', 'permission', 'access', 'blocked'],
            'feature': ['request', 'feature', 'enhancement', 'suggestion'],
            'general': []
        }
        
        category = 'general'
        for cat, keywords in categories.items():
            if any(kw in text for kw in keywords):
                category = cat
                break
        
        priority = 'low'
        if any(word in text for word in ['urgent', 'critical', 'down', 'crash']):
            priority = 'high'
        elif any(word in text for word in ['important', 'soon', 'blocking']):
            priority = 'medium'
        
        return {
            'category': category,
            'subCategory': 'database' if 'database' in text else category,
            'priority': priority,
            'suggestedAssignee': 'devops_team' if category == 'technical' else 'support_team',
            'confidence': 0.85
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Patterns and Workflows

### Complete User Registration Flow

```javascript
// Frontend: Register new user (Admin)
const registerUser = async (userData) => {
  try {
    const response = await axios.post('/api/users', userData);
    
    // Automatically check for initial risk
    const riskResponse = await axios.post(
      `${ML_SERVICE_URL}/api/ml/predict-risk`,
      {
        userId: response.data._id,
        taskCompletionRate: 1.0,
        avgTaskDuration: 3600,
        ticketCount: 0,
        loginFrequency: 0
      }
    );
    
    return { ...response.data, initialRisk: riskResponse.data };
  } catch (error) {
    console.error('Registration failed:', error);
    throw error;
  }
};
```

### Real-time Anomaly Monitoring

```javascript
// Monitor user actions for anomalies
const monitorUserAction = async (action) => {
  try {
    const anomalyCheck = await axios.post(
      `${ML_SERVICE_URL}/api/ml/detect
