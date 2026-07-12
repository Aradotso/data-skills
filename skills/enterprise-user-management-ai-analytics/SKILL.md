---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and support tickets
triggers:
  - "help me build a user management dashboard with AI analytics"
  - "integrate AI risk detection into user management system"
  - "create an enterprise task management system with burnout analysis"
  - "set up JWT authentication for user management app"
  - "implement AI-powered ticket classification and routing"
  - "build a Kanban board with time tracking features"
  - "add anomaly detection to user behavior monitoring"
  - "create admin dashboard with predictive insights"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Enterprise User Management System with AI Analytics, a full-stack application that combines user/task management with machine learning capabilities for risk detection, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

The Enterprise User Management System is a complete solution for:
- **User Management**: Role-based access control, user CRUD operations, authentication via JWT
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support Tickets**: AI-powered classification, routing, and priority management
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts for suspicious activity

The system consists of three main components:
1. **Frontend**: React.js application for users and admins
2. **Backend**: Node.js REST API with MongoDB
3. **ML Service**: FastAPI service with scikit-learn and River for online learning

## Installation

### Prerequisites
- Node.js 14+ and npm
- Python 3.8+
- MongoDB instance (local or cloud)

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env)**
```bash
# backend/.env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**Frontend (.env)**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env)**
```bash
# ml-service/.env
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

## Running the Application

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

## Backend API Structure

### Authentication Endpoints

```javascript
// Register new user
POST /api/auth/register
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
// Returns: { token: "jwt_token", user: {...} }
```

### User Management Endpoints

```javascript
// Get all users (Admin only)
GET /api/users
Headers: { Authorization: "Bearer <token>" }

// Get user by ID
GET /api/users/:id
Headers: { Authorization: "Bearer <token>" }

// Update user
PUT /api/users/:id
Headers: { Authorization: "Bearer <token>" }
{
  "name": "Updated Name",
  "email": "updated@example.com",
  "role": "admin"
}

// Delete user (Admin only)
DELETE /api/users/:id
Headers: { Authorization: "Bearer <token>" }
```

### Task Management Endpoints

```javascript
// Create task
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement feature X",
  "description": "Complete implementation",
  "assignedTo": "userId",
  "status": "todo", // todo, in-progress, done
  "priority": "high",
  "deadline": "2026-05-01T00:00:00Z"
}

// Get user tasks
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }

// Update task status
PUT /api/tasks/:id/status
Headers: { Authorization: "Bearer <token>" }
{
  "status": "in-progress"
}

// Track time on task
POST /api/tasks/:id/time
Headers: { Authorization: "Bearer <token>" }
{
  "duration": 3600 // seconds
}
```

### Support Ticket Endpoints

```javascript
// Create support ticket
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "title": "System not responding",
  "description": "Unable to access dashboard",
  "priority": "high",
  "category": "technical"
}

// Get tickets with AI classification
GET /api/tickets
Headers: { Authorization: "Bearer <token>" }
// Returns tickets with AI-assigned categories and priorities

// Update ticket
PUT /api/tickets/:id
Headers: { Authorization: "Bearer <token>" }
{
  "status": "resolved",
  "resolution": "Fixed authentication issue"
}
```

## ML Service API

### Risk Detection

```python
# POST /api/ml/risk-detection
import requests

def check_user_risk(user_id, activity_data):
    """
    Detect risk based on user behavior
    """
    response = requests.post(
        "http://localhost:8000/api/ml/risk-detection",
        json={
            "userId": user_id,
            "loginAttempts": activity_data["login_attempts"],
            "failedLogins": activity_data["failed_logins"],
            "locationChanges": activity_data["location_changes"],
            "taskCompletionRate": activity_data["completion_rate"],
            "averageSessionDuration": activity_data["avg_session"]
        }
    )
    return response.json()
    # Returns: { "riskScore": 0.75, "riskLevel": "high", "factors": [...] }
```

### Anomaly Detection

```python
# POST /api/ml/anomaly-detection
def detect_anomalies(user_behavior):
    """
    Detect unusual patterns in user behavior
    """
    response = requests.post(
        "http://localhost:8000/api/ml/anomaly-detection",
        json={
            "userId": "user123",
            "features": {
                "loginTime": "03:00",  # Unusual login time
                "dataAccessed": 5000,   # Large amount of data
                "requestsPerMinute": 150,
                "locationIp": "192.168.1.1"
            }
        }
    )
    return response.json()
    # Returns: { "isAnomaly": true, "anomalyScore": 0.92, "reason": "Unusual access pattern" }
```

### Burnout Detection

```python
# POST /api/ml/burnout-detection
def check_burnout_risk(employee_data):
    """
    Analyze workload and predict burnout risk
    """
    response = requests.post(
        "http://localhost:8000/api/ml/burnout-detection",
        json={
            "userId": "user123",
            "hoursWorked": 65,
            "tasksCompleted": 45,
            "overdueItems": 8,
            "weekendWork": 3,
            "consecutiveDaysWorked": 12
        }
    )
    return response.json()
    # Returns: { "burnoutRisk": "high", "score": 0.85, "recommendations": [...] }
```

### Ticket Classification

```python
# POST /api/ml/ticket-classification
def classify_ticket(ticket_content):
    """
    Auto-classify support tickets using AI
    """
    response = requests.post(
        "http://localhost:8000/api/ml/ticket-classification",
        json={
            "title": "Cannot access reports module",
            "description": "Getting 403 error when trying to view reports",
            "userRole": "manager"
        }
    )
    return response.json()
    # Returns: { "category": "access_control", "priority": "high", "assignTo": "security_team" }
```

### Project Delay Prediction

```python
# POST /api/ml/project-insights
def predict_project_delays(project_data):
    """
    Predict if project will be delayed
    """
    response = requests.post(
        "http://localhost:8000/api/ml/project-insights",
        json={
            "projectId": "proj123",
            "totalTasks": 50,
            "completedTasks": 15,
            "daysRemaining": 30,
            "teamSize": 5,
            "averageVelocity": 1.2
        }
    )
    return response.json()
    # Returns: { "delayProbability": 0.68, "estimatedDelay": 10, "recommendations": [...] }
```

## React Frontend Integration

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      // Verify token and load user
      loadUser();
    }
  }, [token]);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data);
    } catch (err) {
      logout();
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
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
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const res = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/user/me`);
    const grouped = {
      todo: res.data.filter(t => t.status === 'todo'),
      inProgress: res.data.filter(t => t.status === 'in-progress'),
      done: res.data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await axios.put(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      status: newStatus
    });
    fetchTasks();
  };

  const onDragStart = (e, taskId, status) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('fromStatus', status);
  };

  const onDrop = async (e, toStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    const fromStatus = e.dataTransfer.getData('fromStatus');
    
    if (fromStatus !== toStatus) {
      await updateTaskStatus(taskId, toStatus);
    }
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDrop={(e) => onDrop(e, status)}
          onDragOver={(e) => e.preventDefault()}
        >
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              draggable
              onDragStart={(e) => onDragStart(e, task._id, status)}
              className="task-card"
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority-${task.priority}`}>
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
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    anomalies: [],
    burnoutAlerts: []
  });

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    // Fetch high-risk users
    const riskRes = await axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/risk-report`);
    
    // Fetch anomalies
    const anomalyRes = await axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/anomalies`);
    
    // Fetch burnout alerts
    const burnoutRes = await axios.get(`${process.env.REACT_APP_ML_SERVICE_URL}/api/ml/burnout-alerts`);

    setAnalytics({
      riskUsers: riskRes.data,
      anomalies: anomalyRes.data,
      burnoutAlerts: burnoutRes.data
    });
  };

  return (
    <div className="admin-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <section className="risk-section">
        <h3>High Risk Users</h3>
        {analytics.riskUsers.map(user => (
          <div key={user.userId} className="alert-card">
            <p><strong>{user.userName}</strong></p>
            <p>Risk Score: {(user.riskScore * 100).toFixed(1)}%</p>
            <p>Factors: {user.factors.join(', ')}</p>
          </div>
        ))}
      </section>

      <section className="anomaly-section">
        <h3>Detected Anomalies</h3>
        {analytics.anomalies.map(anomaly => (
          <div key={anomaly.id} className="alert-card">
            <p><strong>{anomaly.userName}</strong></p>
            <p>Type: {anomaly.reason}</p>
            <p>Score: {(anomaly.anomalyScore * 100).toFixed(1)}%</p>
            <p>Time: {new Date(anomaly.timestamp).toLocaleString()}</p>
          </div>
        ))}
      </section>

      <section className="burnout-section">
        <h3>Burnout Alerts</h3>
        {analytics.burnoutAlerts.map(alert => (
          <div key={alert.userId} className="alert-card">
            <p><strong>{alert.userName}</strong></p>
            <p>Risk Level: {alert.burnoutRisk}</p>
            <p>Hours Worked: {alert.hoursWorked}/week</p>
            <p>Recommendations:</p>
            <ul>
              {alert.recommendations.map((rec, i) => (
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

## Backend Models

### User Model

```javascript
// backend/models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true
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
    enum: ['user', 'admin'],
    default: 'user'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  lastLogin: Date,
  isActive: {
    type: Boolean,
    default: true
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
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// backend/models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: String,
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
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
  deadline: Date,
  timeTracked: {
    type: Number,
    default: 0 // in seconds
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  completedAt: Date
});

module.exports = mongoose.model('Task', taskSchema);
```

### Ticket Model

```javascript
// backend/models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true
  },
  description: {
    type: String,
    required: true
  },
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  status: {
    type: String,
    enum: ['open', 'in-progress', 'resolved', 'closed'],
    default: 'open'
  },
  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'critical']
  },
  category: String, // AI-assigned
  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  resolution: String,
  aiClassification: {
    confidence: Number,
    suggestedPriority: String,
    suggestedCategory: String
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

## ML Service Implementation

### Risk Detection Model

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='./models/risk_model.pkl'):
        self.model_path = model_path
        if os.path.exists(model_path):
            self.model = joblib.load(model_path)
        else:
            self.model = RandomForestClassifier(n_estimators=100)
            self.is_trained = False
    
    def extract_features(self, user_data):
        """
        Extract features from user activity data
        """
        features = [
            user_data.get('loginAttempts', 0),
            user_data.get('failedLogins', 0),
            user_data.get('locationChanges', 0),
            user_data.get('taskCompletionRate', 0),
            user_data.get('averageSessionDuration', 0),
            user_data.get('dataAccessVolume', 0),
            user_data.get('offHoursActivity', 0)
        ]
        return np.array(features).reshape(1, -1)
    
    def predict_risk(self, user_data):
        """
        Predict risk score for a user
        """
        features = self.extract_features(user_data)
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(features)[0][1]
        else:
            # Fallback to rule-based scoring if not trained
            risk_score = self._rule_based_scoring(user_data)
        
        risk_level = self._classify_risk_level(risk_score)
        factors = self._identify_risk_factors(user_data)
        
        return {
            'riskScore': float(risk_score),
            'riskLevel': risk_level,
            'factors': factors
        }
    
    def _rule_based_scoring(self, user_data):
        """
        Fallback rule-based risk scoring
        """
        score = 0.0
        
        if user_data.get('failedLogins', 0) > 5:
            score += 0.3
        if user_data.get('locationChanges', 0) > 3:
            score += 0.2
        if user_data.get('taskCompletionRate', 1.0) < 0.5:
            score += 0.2
        if user_data.get('offHoursActivity', 0) > 10:
            score += 0.3
        
        return min(score, 1.0)
    
    def _classify_risk_level(self, score):
        if score >= 0.75:
            return 'critical'
        elif score >= 0.5:
            return 'high'
        elif score >= 0.25:
            return 'medium'
        else:
            return 'low'
    
    def _identify_risk_factors(self, user_data):
        factors = []
        
        if user_data.get('failedLogins', 0) > 5:
            factors.append('Multiple failed login attempts')
        if user_data.get('locationChanges', 0) > 3:
            factors.append('Unusual location changes')
        if user_data.get('taskCompletionRate', 1.0) < 0.5:
            factors.append('Low task completion rate')
        if user_data.get('offHoursActivity', 0) > 10:
            factors.append('High off-hours activity')
        
        return factors
```

### Burnout Detection

```python
# ml-service/models/burnout_detector.py
class BurnoutDetector:
    def __init__(self):
        self.thresholds = {
            'hours_per_week': 50,
            'consecutive_days': 10,
            'weekend_work_days': 2,
            'overdue_tasks': 5
        }
    
    def analyze(self, employee_data):
        """
        Analyze employee workload and predict burnout risk
        """
        score = 0.0
        factors = []
        
        hours_worked = employee_data.get('hoursWorked', 0)
        if hours_worked > self.thresholds['hours_per_week']:
            score += 0.3
            factors.append(f'Working {hours_worked} hours/week')
        
        consecutive_days = employee_data.get('consecutiveDaysWorked', 0)
        if consecutive_days > self.thresholds['consecutive_days']:
            score += 0.25
            factors.append(f'{consecutive_days} consecutive work days')
        
        weekend_work = employee_data.get('weekendWork', 0)
        if weekend_work > self.thresholds['weekend_work_days']:
            score += 0.2
            factors.append(f'Worked {weekend_work} weekends')
        
        overdue_items = employee_data.get('overdueItems', 0)
        if overdue_items > self.thresholds['overdue_tasks']:
            score += 0.25
            factors.append(f'{overdue_items} overdue tasks')
        
        risk_level = self._classify_burnout_risk(score)
        recommendations = self._generate_recommendations(score, factors)
        
        return {
            'burnoutRisk': risk_level,
            'score': min(score, 1.0),
            'factors': factors,
            'recommendations': recommendations
        }
    
    def _classify_burnout_risk(self, score):
        if score >= 0.7:
            return 'critical'
        elif score >= 0.5:
            return 'high'
        elif score >= 0.3:
            return 'moderate'
        else:
            return 'low'
    
    def _generate_recommendations(self, score, factors):
        recommendations = []
        
        if score >= 0.5:
            recommendations.append('Immediate intervention required - schedule time off')
            recommendations.append('Redistribute workload to team members')
        
        if 'consecutive work days' in ' '.join(factors):
            recommendations.append('Ensure mandatory rest days')
        
        if 'overdue tasks' in ' '.join(factors):
            recommendations.append('Review and reprioritize task assignments')
        
        if 'weekend' in ' '.join(factors):
            recommendations.append('Implement no-weekend-work policy')
        
        return recommendations
```

### FastAPI Main Service

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.risk_detector import RiskDetector
from models.burnout_detector import BurnoutDetector
from models.ticket_classifier import TicketClassifier
import os

app = FastAPI(title="Enterprise User Management ML Service")

# Initialize models
risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()
ticket_classifier = TicketClassifier()

class RiskDetectionRequest(BaseModel):
    userId: str
    loginAttempts: int
    failedLogins: int
    locationChanges: int
    taskCompletionRate: float
    averageSessionDuration: int
    dataAccessVolume: int = 0
    offHoursActivity: int = 0

class BurnoutDetectionRequest(BaseModel):
    userId: str
    hoursWorked: int
    tasksCompleted: int
    overdueItems: int
    weekendWork: int
    consecutiveDaysWorked: int

class TicketClassificationRequest(BaseModel):
    title: str
    description: str
    userRole: str = "user"

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskDetectionRequest):
    """
    Detect security and performance risks based on user behavior
    """
    try:
        result = risk_detector.predict_risk(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutDetectionRequest):
    """
    Analyze employee workload and predict burnout risk
    """
    try:
        result = burnout_detector.analyze(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/ticket-classification")
async def classify_ticket(request: TicketClassificationRequest):
    """
    Automatically classify and route support tickets
    """
    try:
        result = ticket_classifier.classify(request.dict())
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Patterns

### Protected Routes in Backend

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ message: 'No authentication token' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(401).json({ message: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ message: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### Using Middleware in Routes

```javascript
// backend/routes/users.js
const express = require('express');
const router = express.Router();
const { authMiddleware, adminMiddleware } = require('../middleware/auth');
const User = require('../models/User');

// Get all users (Admin only)
router.get('/', authMiddleware, adminMiddleware, async (req, res) => {
  try {
    const users
