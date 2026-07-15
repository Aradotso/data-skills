---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create user management dashboard with ML"
  - "implement risk detection in user system"
  - "build task management with AI insights"
  - "add anomaly detection to user platform"
  - "configure JWT authentication for enterprise app"
  - "deploy user management with AI features"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI/ML capabilities including risk prediction, anomaly detection, burnout analysis, and predictive project insights.

## What This Project Does

The Enterprise User Management System provides:

- **User Management**: Role-based access control, authentication with JWT, user CRUD operations
- **Task Management**: Kanban board (To Do/In Progress/Done), time tracking, task assignment
- **Support Tickets**: Ticket creation, tracking, AI-based classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Admin Dashboard**: Organization analytics, audit logs, user monitoring
- **User Dashboard**: Personal task overview, performance insights, notifications

Built with React.js frontend, Node.js backend, FastAPI ML service, and MongoDB database.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance running

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
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
EOF

npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
EOF

uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "securepass123",
  "role": "user"
}

// Login
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepass123"
}
// Returns: { token: "jwt_token", user: {...} }

// Verify token
GET /api/auth/verify
Headers: { Authorization: "Bearer jwt_token" }
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { Authorization: "Bearer admin_jwt_token" }

// Get user by ID
GET /api/users/:id

// Update user
PUT /api/users/:id
{
  "username": "john.updated",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
```

### Task Management

```javascript
// Create task
POST /api/tasks
{
  "title": "Implement user dashboard",
  "description": "Create React components for user dashboard",
  "assignedTo": "user_id",
  "status": "todo",
  "priority": "high",
  "dueDate": "2026-05-01"
}

// Get user tasks
GET /api/tasks/user/:userId

// Update task status
PUT /api/tasks/:id
{
  "status": "in_progress",
  "timeSpent": 120
}

// Move task in Kanban
PATCH /api/tasks/:id/status
{
  "status": "done"
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
{
  "title": "Unable to access dashboard",
  "description": "Getting 403 error when accessing dashboard",
  "priority": "high",
  "category": "technical"
}

// Get all tickets (Admin)
GET /api/tickets

// Get user tickets
GET /api/tickets/user/:userId

// Update ticket
PUT /api/tickets/:id
{
  "status": "in_progress",
  "assignedTo": "admin_id",
  "response": "Working on this issue"
}
```

## ML Service API Reference

### Risk Prediction

```python
# Predict user risk score
POST /api/ml/risk-prediction
{
  "userId": "user_id",
  "features": {
    "taskCompletionRate": 0.75,
    "averageTaskDelay": 2.5,
    "ticketCount": 3,
    "loginFrequency": 25,
    "lastActivityDays": 1
  }
}
# Returns: { riskScore: 0.35, riskLevel: "low", factors: [...] }
```

### Anomaly Detection

```python
# Detect anomalies in user behavior
POST /api/ml/anomaly-detection
{
  "userId": "user_id",
  "activity": {
    "loginTime": "03:45:00",
    "location": "new_location",
    "tasksModified": 15,
    "dataAccessed": 500
  }
}
# Returns: { isAnomaly: true, score: 0.85, details: "Unusual login time and location" }
```

### Burnout Detection

```python
# Analyze burnout risk
POST /api/ml/burnout-detection
{
  "userId": "user_id",
  "workload": {
    "tasksAssigned": 25,
    "hoursWorked": 55,
    "weekendWork": 12,
    "overtimeDays": 4
  }
}
# Returns: { burnoutRisk: "high", score: 0.78, recommendations: [...] }
```

### Project Delay Prediction

```python
# Predict project delays
POST /api/ml/project-insights
{
  "projectId": "proj_123",
  "metrics": {
    "totalTasks": 50,
    "completedTasks": 20,
    "daysElapsed": 30,
    "estimatedDays": 60,
    "teamSize": 5
  }
}
# Returns: { delayProbability: 0.65, estimatedDelay: 10, suggestions: [...] }
```

### Ticket Classification

```python
# Auto-classify support ticket
POST /api/ml/ticket-classification
{
  "title": "Cannot reset password",
  "description": "Password reset email not received",
  "userHistory": {
    "previousTickets": 2,
    "avgResolutionTime": 24
  }
}
# Returns: { category: "account", priority: "high", suggestedAssignee: "admin_id" }
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
      verifyToken(token);
    } else {
      setLoading(false);
    }
  }, []);

  const verifyToken = async (token) => {
    try {
      const response = await axios.get(`${API_URL}/api/auth/verify`);
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    }
    setLoading(false);
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

### Task Management Component

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
      const grouped = groupTasksByStatus(response.data);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return {
      todo: taskList.filter(t => t.status === 'todo'),
      inProgress: taskList.filter(t => t.status === 'in_progress'),
      done: taskList.filter(t => t.status === 'done')
    };
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}/status`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const startTimer = (taskId) => {
    // Implement time tracking logic
    const startTime = Date.now();
    localStorage.setItem(`task_${taskId}_start`, startTime);
  };

  const stopTimer = async (taskId) => {
    const startTime = localStorage.getItem(`task_${taskId}_start`);
    if (startTime) {
      const duration = Math.floor((Date.now() - startTime) / 1000 / 60);
      await axios.put(`${API_URL}/api/tasks/${taskId}`, {
        timeSpent: duration
      });
      localStorage.removeItem(`task_${taskId}_start`);
      fetchTasks();
    }
  };

  return (
    <div className="kanban-board">
      <Column 
        title="To Do" 
        tasks={tasks.todo} 
        onMove={moveTask}
        onStartTimer={startTimer}
      />
      <Column 
        title="In Progress" 
        tasks={tasks.inProgress} 
        onMove={moveTask}
        onStopTimer={stopTimer}
      />
      <Column 
        title="Done" 
        tasks={tasks.done} 
      />
    </div>
  );
};

const Column = ({ title, tasks, onMove, onStartTimer, onStopTimer }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <TaskCard 
        key={task._id} 
        task={task} 
        onMove={onMove}
        onStartTimer={onStartTimer}
        onStopTimer={onStopTimer}
      />
    ))}
  </div>
);

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    risks: [],
    anomalies: [],
    burnout: [],
    projectInsights: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const [users, tasks, tickets] = await Promise.all([
        axios.get(`${API_URL}/api/users`),
        axios.get(`${API_URL}/api/tasks`),
        axios.get(`${API_URL}/api/tickets`)
      ]);

      // Analyze each user
      const userAnalytics = await Promise.all(
        users.data.map(async (user) => {
          const userTasks = tasks.data.filter(t => t.assignedTo === user._id);
          const userTickets = tickets.data.filter(t => t.createdBy === user._id);

          // Risk prediction
          const riskData = await axios.post(`${ML_API_URL}/api/ml/risk-prediction`, {
            userId: user._id,
            features: calculateUserFeatures(user, userTasks, userTickets)
          });

          // Burnout detection
          const burnoutData = await axios.post(`${ML_API_URL}/api/ml/burnout-detection`, {
            userId: user._id,
            workload: calculateWorkload(userTasks)
          });

          return {
            user,
            risk: riskData.data,
            burnout: burnoutData.data
          };
        })
      );

      setAnalytics({
        risks: userAnalytics.filter(u => u.risk.riskLevel !== 'low'),
        burnout: userAnalytics.filter(u => u.burnout.burnoutRisk === 'high'),
        anomalies: [], // Populate with real-time anomaly detection
        projectInsights: [] // Populate with project delay predictions
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const calculateUserFeatures = (user, tasks, tickets) => {
    const completedTasks = tasks.filter(t => t.status === 'done').length;
    const totalTasks = tasks.length;
    
    return {
      taskCompletionRate: totalTasks > 0 ? completedTasks / totalTasks : 0,
      averageTaskDelay: calculateAverageDelay(tasks),
      ticketCount: tickets.length,
      loginFrequency: 25, // Would be tracked in real system
      lastActivityDays: 1
    };
  };

  const calculateWorkload = (tasks) => {
    return {
      tasksAssigned: tasks.length,
      hoursWorked: tasks.reduce((sum, t) => sum + (t.timeSpent || 0), 0) / 60,
      weekendWork: 0, // Track in real system
      overtimeDays: 0
    };
  };

  const calculateAverageDelay = (tasks) => {
    const delayedTasks = tasks.filter(t => {
      if (t.dueDate && t.completedAt) {
        return new Date(t.completedAt) > new Date(t.dueDate);
      }
      return false;
    });
    
    if (delayedTasks.length === 0) return 0;
    
    const totalDelay = delayedTasks.reduce((sum, t) => {
      const delay = new Date(t.completedAt) - new Date(t.dueDate);
      return sum + (delay / (1000 * 60 * 60 * 24)); // Convert to days
    }, 0);
    
    return totalDelay / delayedTasks.length;
  };

  return (
    <div className="admin-dashboard">
      <h1>AI Analytics Dashboard</h1>
      
      <section className="risk-section">
        <h2>High Risk Users</h2>
        {analytics.risks.map(({ user, risk }) => (
          <div key={user._id} className="alert-card">
            <h3>{user.username}</h3>
            <p>Risk Score: {(risk.riskScore * 100).toFixed(0)}%</p>
            <p>Factors: {risk.factors.join(', ')}</p>
          </div>
        ))}
      </section>

      <section className="burnout-section">
        <h2>Burnout Alerts</h2>
        {analytics.burnout.map(({ user, burnout }) => (
          <div key={user._id} className="alert-card">
            <h3>{user.username}</h3>
            <p>Burnout Score: {(burnout.score * 100).toFixed(0)}%</p>
            <ul>
              {burnout.recommendations.map((rec, i) => (
                <li key={i}>{rec}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AdminDashboard;
```

## Backend Implementation Examples

### User Model (MongoDB Schema)

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'manager'],
    default: 'user'
  },
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  profile: {
    firstName: String,
    lastName: String,
    department: String,
    position: String
  }
});

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password method
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Authentication Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const JWT_SECRET = process.env.JWT_SECRET;

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user || user.status !== 'active') {
      throw new Error();
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid authentication token' });
  }
};

const isAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { auth, isAdmin };
```

### Task Controller

```javascript
// backend/controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = new Task({
      ...req.body,
      createdBy: req.user._id
    });
    await task.save();
    res.status(201).json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ 
      assignedTo: req.params.userId 
    }).populate('assignedTo', 'username email');
    
    res.json(tasks);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

exports.updateTaskStatus = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    
    if (!task) {
      return res.status(404).json({ error: 'Task not found' });
    }

    task.status = req.body.status;
    
    if (req.body.status === 'done') {
      task.completedAt = new Date();
    }

    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};

exports.trackTime = async (req, res) => {
  try {
    const task = await Task.findById(req.params.id);
    task.timeSpent = (task.timeSpent || 0) + req.body.timeSpent;
    await task.save();
    res.json(task);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
};
```

## ML Service Implementation

### Risk Prediction Model

```python
# ml-service/models/risk_predictor.py
import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler
import joblib
import os

class RiskPredictor:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.scaler = StandardScaler()
        self.model = None
        self.load_or_create_model()
    
    def load_or_create_model(self):
        model_file = os.path.join(self.model_path, 'risk_model.pkl')
        scaler_file = os.path.join(self.model_path, 'risk_scaler.pkl')
        
        if os.path.exists(model_file) and os.path.exists(scaler_file):
            self.model = joblib.load(model_file)
            self.scaler = joblib.load(scaler_file)
        else:
            self.model = RandomForestClassifier(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )
            os.makedirs(self.model_path, exist_ok=True)
    
    def extract_features(self, user_data):
        """Extract features from user data"""
        features = [
            user_data.get('taskCompletionRate', 0),
            user_data.get('averageTaskDelay', 0),
            user_data.get('ticketCount', 0),
            user_data.get('loginFrequency', 0),
            user_data.get('lastActivityDays', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict(self, user_data):
        """Predict risk score for user"""
        features = self.extract_features(user_data)
        
        # If model is not trained, return baseline prediction
        if not hasattr(self.model, 'classes_'):
            risk_score = self._calculate_baseline_risk(user_data)
            return {
                'riskScore': float(risk_score),
                'riskLevel': self._get_risk_level(risk_score),
                'factors': self._identify_risk_factors(user_data, risk_score)
            }
        
        # Use trained model
        features_scaled = self.scaler.transform(features)
        risk_prob = self.model.predict_proba(features_scaled)[0][1]
        
        return {
            'riskScore': float(risk_prob),
            'riskLevel': self._get_risk_level(risk_prob),
            'factors': self._identify_risk_factors(user_data, risk_prob)
        }
    
    def _calculate_baseline_risk(self, user_data):
        """Calculate risk based on heuristics"""
        score = 0.0
        
        if user_data.get('taskCompletionRate', 1.0) < 0.7:
            score += 0.3
        if user_data.get('averageTaskDelay', 0) > 3:
            score += 0.2
        if user_data.get('ticketCount', 0) > 5:
            score += 0.2
        if user_data.get('lastActivityDays', 0) > 7:
            score += 0.3
        
        return min(score, 1.0)
    
    def _get_risk_level(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.6:
            return 'medium'
        else:
            return 'high'
    
    def _identify_risk_factors(self, user_data, score):
        factors = []
        
        if user_data.get('taskCompletionRate', 1.0) < 0.7:
            factors.append('Low task completion rate')
        if user_data.get('averageTaskDelay', 0) > 3:
            factors.append('High average task delay')
        if user_data.get('ticketCount', 0) > 5:
            factors.append('Multiple support tickets')
        if user_data.get('lastActivityDays', 0) > 7:
            factors.append('Inactive for extended period')
        
        return factors if factors else ['No significant risk factors']
    
    def train(self, X, y):
        """Train the model with new data"""
        X_scaled = self.scaler.fit_transform(X)
        self.model.fit(X_scaled, y)
        
        # Save model
        joblib.dump(self.model, os.path.join(self.model_path, 'risk_model.pkl'))
        joblib.dump(self.scaler, os.path.join(self.model_path, 'risk_scaler.pkl'))
```

### FastAPI Endpoints

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, List
from models.risk_predictor import RiskPredictor
from models.anomaly_detector import AnomalyDetector
from models.burnout_analyzer import BurnoutAnalyzer

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_predictor = RiskPredictor()
anomaly_detector = AnomalyDetector()
burnout_analyzer = BurnoutAnalyzer()

class RiskPredictionRequest(BaseModel):
    userId: str
    features: Dict

class AnomalyDetectionRequest(BaseModel):
    userId: str
    activity: Dict

class BurnoutDetectionRequest(BaseModel):
    userId: str
    workload: Dict

@app.post("/api/ml/risk-prediction")
async def predict_risk(request: RiskPredictionRequest):
    try:
        result = risk_predictor.predict(request.features)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/anomaly-detection")
async def detect_anomaly(request: AnomalyDetectionRequest):
    try:
        result = anomaly_detector.detect(request.activity)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    try:
        result = burnout_analyzer.analyze(request.workload)
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/ticket-classification")
async def classify_ticket(ticket_data: Dict):
    try:
        # Simple rule-based classification
        title_lower = ticket_data.get('title', '').lower()
        description_lower = ticket_data.get('description', '').lower()
        
        category = 'general'
        priority = 'medium'
        
        # Categorization logic
        if any(word in title_lower or word in description_lower 
               for word in ['password', 'login', 'access', 'account']):
            category = 'account'
            priority = 'high'
        elif any(word in title_lower or word in description_lower 
                 for word in ['bug', 'error', 'crash', 'broken']):
            category = 'technical'
            priority = 'high'
        elif any(word in title_lower or word in description_lower 
                 for word in ['feature', 'request', 'enhancement']):
            category = 'feature_request'
            priority = 'low'
        
        return {
            'category': category,
            'priority': priority,
            'suggestedAssignee': None  # Could be enhanced with assignment logic
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ML Service"}
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=info
ANOMALY_THRESHOLD=0.7
RISK_UPDATE_INTERVAL=3600
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_SOCKET_URL=http://localhost:5000
REACT_APP_ENV=development
```

### Database Indexes

```javascript
// Create indexes for performance
db.users.createIndex({ email: 1 }, { unique
