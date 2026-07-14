---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, risk detection, and task management using React, Node.js, FastAPI, and MongoDB
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "integrate ML service for risk detection"
  - "build admin panel with user management"
  - "add AI-powered ticket classification"
  - "configure JWT authentication for user system"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that provides centralized user, task, and ticket management with integrated AI capabilities. The system includes:

- **Frontend**: React.js-based user and admin dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI engine for risk detection, anomaly detection, burnout analysis, and ticket classification
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
- MongoDB (local or cloud instance)

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
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
```

Start backend:

```bash
npm start
# or for development with hot reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```env
MONGODB_URI=your_mongodb_connection_string
MODEL_PATH=./models
BACKEND_URL=http://localhost:5000
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
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Reference

### Authentication

```javascript
// Register new user
POST /api/auth/register
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securePassword123",
  "role": "user"
}

// Login
POST /api/auth/login
Content-Type: application/json

{
  "email": "john@example.com",
  "password": "securePassword123"
}

// Response includes JWT token
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "user_id",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Authorization: Bearer {token}

// Get user by ID
GET /api/users/:id
Authorization: Bearer {token}

// Update user
PUT /api/users/:id
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "admin",
  "status": "active"
}

// Delete user
DELETE /api/users/:id
Authorization: Bearer {token}
```

### Task Management

```javascript
// Create task
POST /api/tasks
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Implement authentication",
  "description": "Add JWT-based auth system",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Get user tasks
GET /api/tasks/user/:userId
Authorization: Bearer {token}

// Update task status
PATCH /api/tasks/:id/status
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "in-progress"
}

// Track time
POST /api/tasks/:id/time
Authorization: Bearer {token}
Content-Type: application/json

{
  "duration": 3600,
  "date": "2026-04-15"
}
```

### Ticket Management

```javascript
// Create support ticket
POST /api/tickets
Authorization: Bearer {token}
Content-Type: application/json

{
  "title": "Login issue",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}

// Get all tickets
GET /api/tickets
Authorization: Bearer {token}

// Update ticket
PATCH /api/tickets/:id
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "resolved",
  "resolution": "Password reset sent"
}
```

## ML Service API Reference

### Risk Detection

```javascript
// Analyze user risk
POST /api/ml/risk-detection
Content-Type: application/json

{
  "userId": "user_id",
  "loginAttempts": 5,
  "taskCompletionRate": 0.45,
  "lastActivityDays": 15,
  "ticketCount": 8
}

// Response
{
  "riskScore": 0.78,
  "riskLevel": "high",
  "factors": [
    "Low task completion rate",
    "High ticket count",
    "Inactive for extended period"
  ],
  "recommendations": [
    "Schedule check-in meeting",
    "Review workload distribution"
  ]
}
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
POST /api/ml/anomaly-detection
Content-Type: application/json

{
  "userId": "user_id",
  "activities": [
    {
      "timestamp": "2026-04-15T10:30:00Z",
      "action": "login",
      "ipAddress": "192.168.1.100"
    },
    {
      "timestamp": "2026-04-15T10:35:00Z",
      "action": "data_export",
      "recordCount": 5000
    }
  ]
}

// Response
{
  "anomalyDetected": true,
  "anomalyScore": 0.89,
  "alerts": [
    "Unusual data export volume",
    "Access from new IP address"
  ]
}
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
POST /api/ml/burnout-detection
Content-Type: application/json

{
  "userId": "user_id",
  "workHoursPerWeek": 65,
  "overtimeHours": 25,
  "tasksCompleted": 15,
  "tasksOverdue": 8,
  "ticketsRaised": 6,
  "daysWithoutBreak": 12
}

// Response
{
  "burnoutRisk": "high",
  "burnoutScore": 0.82,
  "indicators": [
    "Excessive work hours",
    "High overdue task count",
    "Extended period without breaks"
  ],
  "suggestions": [
    "Redistribute workload",
    "Enforce mandatory time off",
    "Reduce task assignments"
  ]
}
```

### Ticket Classification

```javascript
// Classify support ticket
POST /api/ml/ticket-classification
Content-Type: application/json

{
  "title": "Cannot access database",
  "description": "Getting connection timeout errors when trying to connect to production database"
}

// Response
{
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "database_admin_team",
  "estimatedResolutionTime": "2-4 hours",
  "confidence": 0.92
}
```

### Predictive Analytics

```javascript
// Predict project delays
POST /api/ml/project-insights
Content-Type: application/json

{
  "projectId": "project_id",
  "totalTasks": 50,
  "completedTasks": 20,
  "overdueTasks": 5,
  "averageCompletionTime": 3.5,
  "remainingDays": 30,
  "teamSize": 5
}

// Response
{
  "delayProbability": 0.67,
  "estimatedDelay": 7,
  "riskFactors": [
    "Current completion rate below target",
    "High overdue task percentage"
  ],
  "recommendations": [
    "Add 2 more team members",
    "Reduce scope by 10 tasks",
    "Increase daily standup frequency"
  ]
}
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
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`);
      setUser(response.data.user);
    } catch (error) {
      localStorage.removeItem('token');
      delete axios.defaults.headers.common['Authorization'];
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// src/components/TaskDashboard.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/tasks/user/me`);
      const grouped = groupTasksByStatus(response.data.tasks);
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const groupTasksByStatus = (taskList) => {
    return {
      todo: taskList.filter(t => t.status === 'todo'),
      inProgress: taskList.filter(t => t.status === 'in-progress'),
      done: taskList.filter(t => t.status === 'done')
    };
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    const currentStatus = e.dataTransfer.getData('currentStatus');
    
    if (currentStatus !== newStatus) {
      updateTaskStatus(taskId, newStatus);
    }
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="task-dashboard">
      <div className="kanban-board">
        {['todo', 'inProgress', 'done'].map(status => (
          <div
            key={status}
            className="kanban-column"
            onDrop={(e) => onDrop(e, status)}
            onDragOver={onDragOver}
          >
            <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
            {tasks[status].map(task => (
              <div
                key={task._id}
                className="task-card"
                draggable
                onDragStart={(e) => onDragStart(e, task._id, status)}
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
    </div>
  );
};

export default TaskDashboard;
```

### AI Analytics Integration

```javascript
// src/services/aiAnalytics.js
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

export const analyzeUserRisk = async (userId, userData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/risk-detection`, {
      userId,
      ...userData
    });
    return response.data;
  } catch (error) {
    console.error('Risk analysis failed:', error);
    throw error;
  }
};

export const detectBurnout = async (userId, workloadData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/burnout-detection`, {
      userId,
      ...workloadData
    });
    return response.data;
  } catch (error) {
    console.error('Burnout detection failed:', error);
    throw error;
  }
};

export const classifyTicket = async (ticketData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/ticket-classification`, ticketData);
    return response.data;
  } catch (error) {
    console.error('Ticket classification failed:', error);
    throw error;
  }
};

export const getProjectInsights = async (projectData) => {
  try {
    const response = await axios.post(`${ML_API_URL}/api/ml/project-insights`, projectData);
    return response.data;
  } catch (error) {
    console.error('Project insights failed:', error);
    throw error;
  }
};
```

### Admin Analytics Dashboard

```javascript
// src/components/AdminDashboard.js
import React, { useState, useEffect } from 'react';
import { analyzeUserRisk, detectBurnout } from '../services/aiAnalytics';
import axios from 'axios';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    fetchUsersAndAnalyze();
  }, []);

  const fetchUsersAndAnalyze = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/users`);
      setUsers(response.data.users);
      
      // Analyze each user
      const analysisPromises = response.data.users.map(async (user) => {
        const [riskData, burnoutData] = await Promise.all([
          analyzeUserRisk(user._id, {
            loginAttempts: user.loginAttempts || 0,
            taskCompletionRate: user.taskCompletionRate || 0,
            lastActivityDays: calculateDaysSinceActivity(user.lastActivity),
            ticketCount: user.ticketCount || 0
          }),
          detectBurnout(user._id, {
            workHoursPerWeek: user.workHoursPerWeek || 40,
            overtimeHours: user.overtimeHours || 0,
            tasksCompleted: user.tasksCompleted || 0,
            tasksOverdue: user.tasksOverdue || 0,
            ticketsRaised: user.ticketsRaised || 0,
            daysWithoutBreak: user.daysWithoutBreak || 0
          })
        ]);
        
        return {
          userId: user._id,
          riskData,
          burnoutData
        };
      });
      
      const results = await Promise.all(analysisPromises);
      const analyticsMap = {};
      const newAlerts = [];
      
      results.forEach(result => {
        analyticsMap[result.userId] = result;
        
        if (result.riskData.riskLevel === 'high') {
          newAlerts.push({
            type: 'risk',
            userId: result.userId,
            message: `High risk detected for user ${result.userId}`
          });
        }
        
        if (result.burnoutData.burnoutRisk === 'high') {
          newAlerts.push({
            type: 'burnout',
            userId: result.userId,
            message: `Burnout risk detected for user ${result.userId}`
          });
        }
      });
      
      setAnalytics(analyticsMap);
      setAlerts(newAlerts);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const calculateDaysSinceActivity = (lastActivity) => {
    if (!lastActivity) return 999;
    const diff = Date.now() - new Date(lastActivity).getTime();
    return Math.floor(diff / (1000 * 60 * 60 * 24));
  };

  return (
    <div className="admin-dashboard">
      <h2>Admin Analytics Dashboard</h2>
      
      {alerts.length > 0 && (
        <div className="alerts-section">
          <h3>Alerts</h3>
          {alerts.map((alert, index) => (
            <div key={index} className={`alert alert-${alert.type}`}>
              {alert.message}
            </div>
          ))}
        </div>
      )}
      
      <div className="users-grid">
        {users.map(user => {
          const userAnalytics = analytics[user._id];
          return (
            <div key={user._id} className="user-card">
              <h4>{user.name}</h4>
              <p>{user.email}</p>
              {userAnalytics && (
                <>
                  <div className="risk-indicator">
                    Risk: <span className={`level-${userAnalytics.riskData.riskLevel}`}>
                      {userAnalytics.riskData.riskLevel}
                    </span>
                  </div>
                  <div className="burnout-indicator">
                    Burnout: <span className={`level-${userAnalytics.burnoutData.burnoutRisk}`}>
                      {userAnalytics.burnoutData.burnoutRisk}
                    </span>
                  </div>
                </>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Configuration

### Backend Models

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
  status: {
    type: String,
    enum: ['active', 'inactive', 'suspended'],
    default: 'active'
  },
  lastActivity: Date,
  createdAt: {
    type: Date,
    default: Date.now
  }
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
  assignedBy: {
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
  dueDate: Date,
  timeTracked: {
    type: Number,
    default: 0
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

module.exports = mongoose.model('Task', taskSchema);
```

### ML Service Models

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='models/risk_model.pkl'):
        self.model_path = model_path
        self.model = self._load_or_create_model()
        
    def _load_or_create_model(self):
        if os.path.exists(self.model_path):
            return joblib.load(self.model_path)
        else:
            model = RandomForestClassifier(n_estimators=100, random_state=42)
            os.makedirs(os.path.dirname(self.model_path), exist_ok=True)
            return model
    
    def predict_risk(self, features):
        """
        Features: [login_attempts, task_completion_rate, last_activity_days, ticket_count]
        """
        X = np.array(features).reshape(1, -1)
        
        # Normalize features
        X_norm = self._normalize_features(X)
        
        # If model hasn't been trained, use rule-based approach
        if not hasattr(self.model, 'classes_'):
            return self._rule_based_risk(features)
        
        risk_score = self.model.predict_proba(X_norm)[0][1]
        risk_level = self._categorize_risk(risk_score)
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'factors': self._identify_factors(features),
            'recommendations': self._get_recommendations(risk_level, features)
        }
    
    def _normalize_features(self, X):
        # Simple normalization
        return X / np.array([10, 1, 30, 20])
    
    def _rule_based_risk(self, features):
        login_attempts, completion_rate, inactive_days, ticket_count = features
        
        risk_score = 0
        if login_attempts > 5:
            risk_score += 0.3
        if completion_rate < 0.5:
            risk_score += 0.3
        if inactive_days > 7:
            risk_score += 0.2
        if ticket_count > 5:
            risk_score += 0.2
        
        risk_level = self._categorize_risk(risk_score)
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'factors': self._identify_factors(features),
            'recommendations': self._get_recommendations(risk_level, features)
        }
    
    def _categorize_risk(self, score):
        if score < 0.3:
            return 'low'
        elif score < 0.6:
            return 'medium'
        else:
            return 'high'
    
    def _identify_factors(self, features):
        factors = []
        login_attempts, completion_rate, inactive_days, ticket_count = features
        
        if login_attempts > 5:
            factors.append('High number of login attempts')
        if completion_rate < 0.5:
            factors.append('Low task completion rate')
        if inactive_days > 7:
            factors.append('Inactive for extended period')
        if ticket_count > 5:
            factors.append('High ticket count')
        
        return factors
    
    def _get_recommendations(self, risk_level, features):
        recommendations = []
        _, completion_rate, inactive_days, _ = features
        
        if risk_level in ['medium', 'high']:
            recommendations.append('Schedule check-in meeting')
            
        if completion_rate < 0.5:
            recommendations.append('Review workload distribution')
            
        if inactive_days > 7:
            recommendations.append('Send activity reminder')
        
        return recommendations
    
    def save_model(self):
        joblib.dump(self.model, self.model_path)
```

```python
# ml-service/models/burnout_detector.py
import numpy as np

class BurnoutDetector:
    def analyze(self, data):
        work_hours = data.get('workHoursPerWeek', 40)
        overtime = data.get('overtimeHours', 0)
        completed = data.get('tasksCompleted', 0)
        overdue = data.get('tasksOverdue', 0)
        tickets = data.get('ticketsRaised', 0)
        days_no_break = data.get('daysWithoutBreak', 0)
        
        # Calculate burnout score
        score = 0
        
        # Work hours factor
        if work_hours > 60:
            score += 0.3
        elif work_hours > 50:
            score += 0.2
        elif work_hours > 45:
            score += 0.1
        
        # Overtime factor
        if overtime > 20:
            score += 0.2
        elif overtime > 10:
            score += 0.1
        
        # Task management factor
        if overdue > 0:
            overdue_ratio = overdue / max(completed + overdue, 1)
            score += min(overdue_ratio * 0.3, 0.3)
        
        # Ticket frequency factor
        if tickets > 5:
            score += 0.1
        
        # Break factor
        if days_no_break > 10:
            score += 0.2
        elif days_no_break > 7:
            score += 0.1
        
        # Categorize risk
        if score < 0.3:
            risk_level = 'low'
        elif score < 0.6:
            risk_level = 'medium'
        else:
            risk_level = 'high'
        
        return {
            'burnoutRisk': risk_level,
            'burnoutScore': float(score),
            'indicators': self._get_indicators(work_hours, overtime, overdue, days_no_break),
            'suggestions': self._get_suggestions(risk_level, work_hours, overdue, days_no_break)
        }
    
    def _get_indicators(self, work_hours, overtime, overdue, days_no_break):
        indicators = []
        
        if work_hours > 50:
            indicators.append('Excessive work hours')
        if overtime > 10:
            indicators.append('High overtime hours')
        if overdue > 3:
            indicators.append('High overdue task count')
        if days_no_break > 7:
            indicators.append('Extended period without breaks')
        
        return indicators
    
    def _get_suggestions(self, risk_level, work_hours, overdue, days_no_break):
        suggestions = []
        
        if risk_level in ['medium', 'high']:
            suggestions.append('Redistribute workload')
            
        if work_hours > 50:
            suggestions.append('Reduce weekly work hours')
            
        if overdue > 3:
            suggestions.append('Reduce task assignments')
            
        if days_no_break > 7:
            suggestions.append('Enforce mandatory time off')
        
        return suggestions
```

### ML Service Main Application

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict
from models.risk_detector import RiskDetector
from models.burnout_detector import BurnoutDetector
import uvicorn

app = FastAPI(title="Enterprise AI Analytics Service")

risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()

class RiskAnalysisRequest(BaseModel):
    userId: str
    loginAttempts: int
    taskCompletionRate: float
    lastActivityDays: int
    ticketCount: int

class BurnoutAnalysisRequest(BaseModel):
    userId: str
    workHoursPerWeek: float
    overtimeHours: float
    tasksCompleted: int
    tasksOverdue: int
    ticketsRaised: int
    daysWithoutBreak: int

class TicketClassificationRequest(BaseModel):
    title: str
    description: str

class ProjectInsightsRequest(BaseModel):
    projectId: str
    totalTasks: int
    completedTasks: int
    overdueTasks: int
    averageCompletionTime: float
