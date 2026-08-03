---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with AI"
  - "create user dashboard with task tracking"
  - "add AI ticket classification system"
  - "build admin dashboard with analytics"
  - "configure AI-powered risk detection"
  - "implement burnout detection for users"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI analytics. The system provides intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project management capabilities.

## What This Project Does

The Enterprise User Management System combines traditional CRUD operations for user and task management with AI-driven analytics to:

- Manage users with role-based access control (Admin/User)
- Track tasks using Kanban boards with time tracking
- Handle support ticket routing and classification
- Detect anomalies and security risks using ML models
- Predict employee burnout based on workload patterns
- Provide predictive insights for project delays
- Generate analytics dashboards for organizational metrics

**Stack**: React.js frontend, Node.js backend, FastAPI ML service, MongoDB database

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance running locally or remotely

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
MONGODB_URI=mongodb://localhost:27017/enterprise-users
JWT_SECRET=your_jwt_secret_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
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

Create `.env` file in `ml-service/`:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
MONGODB_URI=mongodb://localhost:27017/enterprise-users
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

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Application runs at: `http://localhost:3000`

## Key API Endpoints

### Authentication (Backend)

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@company.com",
  "password": "securepass123",
  "role": "user" // or "admin"
}

// Login
POST /api/auth/login
{
  "email": "john@company.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }
```

### User Management (Backend)

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer <token>" }
{
  "username": "updated.name",
  "role": "admin"
}

// Delete user (Admin only)
DELETE /api/users/:id
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement login feature",
  "description": "Add JWT authentication",
  "assignedTo": "user_id",
  "priority": "high",
  "status": "todo", // todo, in-progress, done
  "dueDate": "2026-05-01"
}

// Update task status
PATCH /api/tasks/:id/status
{
  "status": "in-progress"
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
Headers: { Authorization: "Bearer <token>" }
{
  "subject": "Cannot access reports",
  "description": "Getting 403 error when accessing analytics",
  "priority": "medium"
}

// AI classifies and routes automatically
```

### AI Analytics Endpoints (ML Service)

```python
# Risk prediction
POST /api/ml/predict-risk
{
  "user_id": "user123",
  "login_attempts": 5,
  "failed_logins": 2,
  "unusual_hours": true,
  "data_access_volume": 150
}
# Returns: { "risk_score": 0.75, "risk_level": "high" }

# Burnout detection
POST /api/ml/detect-burnout
{
  "user_id": "user123",
  "tasks_completed": 45,
  "avg_hours_per_day": 11.5,
  "missed_deadlines": 3,
  "ticket_volume": 12
}
# Returns: { "burnout_probability": 0.82, "recommendation": "immediate_intervention" }

# Ticket classification
POST /api/ml/classify-ticket
{
  "subject": "Cannot access database",
  "description": "Getting connection timeout errors"
}
# Returns: { "category": "technical", "priority": "high", "department": "IT" }

# Anomaly detection
POST /api/ml/detect-anomaly
{
  "user_id": "user123",
  "activity_data": {
    "login_time": "03:45 AM",
    "location": "unknown",
    "data_downloaded": 500 // MB
  }
}
# Returns: { "is_anomaly": true, "confidence": 0.91 }
```

## Frontend Usage Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/login`,
      { email, password }
    );
    setToken(response.data.token);
    setUser(response.data.user);
    localStorage.setItem('token', response.data.token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// src/components/TaskBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    
    const grouped = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(column => (
        <div key={column} className="kanban-column">
          <h3>{column.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[column].map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onStatusChange={updateTaskStatus}
            />
          ))}
        </div>
      ))}
    </div>
  );
};
```

### AI Analytics Integration

```javascript
// src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [riskUsers, setRiskUsers] = useState([]);
  const [burnoutAlerts, setBurnoutAlerts] = useState([]);

  useEffect(() => {
    checkRiskUsers();
    checkBurnout();
  }, []);

  const checkRiskUsers = async () => {
    const users = await fetchAllUsers();
    const riskChecks = await Promise.all(
      users.map(user => 
        axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`,
          {
            user_id: user._id,
            login_attempts: user.loginAttempts,
            failed_logins: user.failedLogins,
            unusual_hours: user.unusualActivity
          }
        )
      )
    );
    
    const highRisk = riskChecks
      .map((r, i) => ({ ...users[i], ...r.data }))
      .filter(u => u.risk_level === 'high');
    
    setRiskUsers(highRisk);
  };

  const checkBurnout = async () => {
    const users = await fetchAllUsers();
    const burnoutChecks = await Promise.all(
      users.map(user =>
        axios.post(
          `${process.env.REACT_APP_ML_API_URL}/api/ml/detect-burnout`,
          {
            user_id: user._id,
            tasks_completed: user.tasksCompleted,
            avg_hours_per_day: user.avgHours,
            missed_deadlines: user.missedDeadlines
          }
        )
      )
    );

    const atRisk = burnoutChecks
      .map((r, i) => ({ ...users[i], ...r.data }))
      .filter(u => u.burnout_probability > 0.7);

    setBurnoutAlerts(atRisk);
  };

  return (
    <div className="analytics-dashboard">
      <section className="risk-alerts">
        <h2>High Risk Users</h2>
        {riskUsers.map(user => (
          <div key={user._id} className="alert-card">
            <p>{user.username} - Risk Score: {user.risk_score}</p>
          </div>
        ))}
      </section>
      
      <section className="burnout-alerts">
        <h2>Burnout Risk</h2>
        {burnoutAlerts.map(user => (
          <div key={user._id} className="alert-card">
            <p>{user.username} - Probability: {user.burnout_probability}</p>
            <p>Action: {user.recommendation}</p>
          </div>
        ))}
      </section>
    </div>
  );
};
```

## Backend API Implementation

### User Controller

```javascript
// backend/controllers/userController.js
const User = require('../models/User');
const jwt = require('jsonwebtoken');

exports.register = async (req, res) => {
  try {
    const { username, email, password, role } = req.body;
    
    const existingUser = await User.findOne({ email });
    if (existingUser) {
      return res.status(400).json({ message: 'User already exists' });
    }

    const user = new User({ username, email, password, role });
    await user.save();

    const token = jwt.sign(
      { userId: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '7d' }
    );

    res.status(201).json({ token, user: user.toJSON() });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.getAllUsers = async (req, res) => {
  try {
    const users = await User.find().select('-password');
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateUser = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;
    
    const user = await User.findByIdAndUpdate(id, updates, { new: true })
      .select('-password');
    
    if (!user) {
      return res.status(404).json({ message: 'User not found' });
    }
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user.userId
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;
    
    const task = await Task.findByIdAndUpdate(
      id,
      { status, updatedAt: Date.now() },
      { new: true }
    );
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const { id } = req.params;
    const { duration } = req.body;
    
    const task = await Task.findById(id);
    task.timeSpent = (task.timeSpent || 0) + duration;
    await task.save();
    
    res.json(task);
  } catch (error) {
    res.status(500).json({ message: error.message });
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
    return res.status(401).json({ message: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

exports.requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
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
    def __init__(self):
        model_path = os.getenv('MODEL_PATH', './models')
        try:
            self.model = joblib.load(f'{model_path}/risk_model.pkl')
        except:
            self.model = RandomForestClassifier(n_estimators=100)
            self._train_initial_model()
    
    def _train_initial_model(self):
        # Synthetic training data for initial model
        X_train = np.random.rand(1000, 4)
        y_train = (X_train[:, 0] + X_train[:, 1] * 2 > 1.5).astype(int)
        self.model.fit(X_train, y_train)
    
    def predict(self, features):
        """
        features: dict with keys:
          - login_attempts
          - failed_logins
          - unusual_hours (bool)
          - data_access_volume
        """
        X = np.array([[
            features['login_attempts'],
            features['failed_logins'],
            1 if features['unusual_hours'] else 0,
            features['data_access_volume']
        ]])
        
        risk_score = self.model.predict_proba(X)[0][1]
        
        if risk_score > 0.7:
            risk_level = 'high'
        elif risk_score > 0.4:
            risk_level = 'medium'
        else:
            risk_level = 'low'
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level
        }
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
import numpy as np
from river import tree, compose, preprocessing

class BurnoutDetector:
    def __init__(self):
        # Online learning model using River
        self.model = compose.Pipeline(
            preprocessing.StandardScaler(),
            tree.HoeffdingAdaptiveTreeRegressor()
        )
        self.trained = False
    
    def predict(self, features):
        """
        features: dict with keys:
          - tasks_completed
          - avg_hours_per_day
          - missed_deadlines
          - ticket_volume
        """
        x = {
            'tasks': features['tasks_completed'],
            'hours': features['avg_hours_per_day'],
            'missed': features['missed_deadlines'],
            'tickets': features['ticket_volume']
        }
        
        # Simple heuristic if model not trained
        if not self.trained:
            burnout_score = (
                min(features['avg_hours_per_day'] / 12, 1.0) * 0.4 +
                min(features['missed_deadlines'] / 10, 1.0) * 0.3 +
                min(features['ticket_volume'] / 20, 1.0) * 0.3
            )
        else:
            burnout_score = self.model.predict_one(x)
        
        burnout_score = max(0, min(1, burnout_score))
        
        if burnout_score > 0.8:
            recommendation = 'immediate_intervention'
        elif burnout_score > 0.6:
            recommendation = 'reduce_workload'
        elif burnout_score > 0.4:
            recommendation = 'monitor_closely'
        else:
            recommendation = 'normal'
        
        return {
            'burnout_probability': float(burnout_score),
            'recommendation': recommendation
        }
    
    def learn(self, features, outcome):
        """Update model with new data"""
        x = {
            'tasks': features['tasks_completed'],
            'hours': features['avg_hours_per_day'],
            'missed': features['missed_deadlines'],
            'tickets': features['ticket_volume']
        }
        self.model.learn_one(x, outcome)
        self.trained = True
```

### FastAPI Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import os

app = FastAPI(title="Enterprise User Management ML Service")

risk_predictor = RiskPredictor()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskPredictionRequest(BaseModel):
    user_id: str
    login_attempts: int
    failed_logins: int
    unusual_hours: bool
    data_access_volume: int

class BurnoutDetectionRequest(BaseModel):
    user_id: str
    tasks_completed: int
    avg_hours_per_day: float
    missed_deadlines: int
    ticket_volume: int

class TicketClassificationRequest(BaseModel):
    subject: str
    description: str

@app.post("/api/ml/predict-risk")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict({
            'login_attempts': request.login_attempts,
            'failed_logins': request.failed_logins,
            'unusual_hours': request.unusual_hours,
            'data_access_volume': request.data_access_volume
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/detect-burnout")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        result = burnout_detector.predict({
            'tasks_completed': request.tasks_completed,
            'avg_hours_per_day': request.avg_hours_per_day,
            'missed_deadlines': request.missed_deadlines,
            'ticket_volume': request.ticket_volume
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketClassificationRequest):
    try:
        result = ticket_classifier.classify(
            request.subject,
            request.description
        )
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-analytics"}
```

### Ticket Classification

```python
# ml-service/models/ticket_classifier.py
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
import re

class TicketClassifier:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=100)
        self.category_model = MultinomialNB()
        self.priority_model = MultinomialNB()
        self._initialize_models()
    
    def _initialize_models(self):
        # Sample training data
        texts = [
            "cannot login database connection error",
            "need access to admin panel",
            "bug in reporting dashboard",
            "request new software license"
        ]
        categories = [0, 1, 0, 1]  # 0: technical, 1: administrative
        
        X = self.vectorizer.fit_transform(texts)
        self.category_model.fit(X, categories)
        self.priority_model.fit(X, [1, 0, 1, 0])  # 1: high, 0: medium
    
    def classify(self, subject, description):
        text = f"{subject} {description}".lower()
        text = re.sub(r'[^a-z\s]', '', text)
        
        X = self.vectorizer.transform([text])
        
        category_pred = self.category_model.predict(X)[0]
        priority_pred = self.priority_model.predict(X)[0]
        
        category = 'technical' if category_pred == 0 else 'administrative'
        priority = 'high' if priority_pred == 1 else 'medium'
        
        # Keyword-based department routing
        if any(word in text for word in ['login', 'access', 'permission']):
            department = 'IT Security'
        elif any(word in text for word in ['bug', 'error', 'crash']):
            department = 'Development'
        elif any(word in text for word in ['license', 'purchase', 'invoice']):
            department = 'Finance'
        else:
            department = 'IT Support'
        
        return {
            'category': category,
            'priority': priority,
            'department': department
        }
```

## Configuration

### MongoDB Models

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  createdAt: { type: Date, default: Date.now },
  loginAttempts: { type: Number, default: 0 },
  failedLogins: { type: Number, default: 0 },
  unusualActivity: { type: Boolean, default: false }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
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
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
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
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

## Common Patterns

### Protected Routes in React

```javascript
// src/components/ProtectedRoute.js
import { useContext } from 'react';
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

### API Client with Axios

```javascript
// src/utils/apiClient.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

apiClient.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

apiClient.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import apiClient from '../utils/apiClient';

const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await apiClient.get(`/api/notifications/${userId}`);
      setNotifications(response.data);
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000); // Poll every 30s

    return () => clearInterval(interval);
  }, [userId]);

  const markAsRead = async (notificationId) => {
    await apiClient.patch(`/api/notifications/${notificationId}/read`);
    setNotifications(prev => 
      prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
};

export default useNotifications;
```

## Troubleshooting

### JWT Token Expiration Issues

If users are logged out unexpectedly:

```javascript
// Implement token refresh logic
const refreshToken = async () => {
  try {
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/api/auth/refresh`,
      {},
      { headers: { Authorization: `Bearer ${oldToken}` } }
    );
    localStorage.setItem('token', response.data.token);
    return response.data.token;
  } catch (error) {
    // Force logout
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};
```

### MongoDB Connection Errors

Ensure MongoDB is running and connection string is correct:

```bash
# Check MongoDB status
sudo systemctl status mongod

# Restart if needed
sudo systemctl restart mongod

# Test connection
mongo --eval "db.adminCommand('ping')"
```

### ML Service Not Loading Models

If models fail to load, initialize them:

```python
# ml-service/scripts/initialize_models.py
from models.risk_predictor import RiskPredictor
from models.burnout_detector import BurnoutDetector
import joblib
import os

def initialize():
    os.makedirs('./models', exist_ok=True)
    
    risk_model = RiskPredictor()
    joblib.dump(risk_model.model, './models/risk_model.pkl')
    
    print("Models initialized successfully")

if __name__ == "__main__":
    initialize()
```

Run: `python ml-service/scripts/initialize_models.py`

### CORS Issues

If
