---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with AI insights"
  - "build task tracking system with anomaly detection"
  - "integrate AI analytics for user behavior monitoring"
  - "develop enterprise admin panel with ML predictions"
  - "add risk detection to user management app"
  - "configure FastAPI ML service for user analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System with AI Analytics is a full-stack application that combines traditional user/task management with machine learning capabilities. It provides:

- **User & Admin Dashboards**: Role-based access for users and administrators
- **Task Management**: Kanban-style boards with time tracking
- **Support Ticket System**: AI-powered ticket classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Security**: JWT-based authentication with audit logging

The system consists of three main components:
1. **Frontend** (React.js) - User interface
2. **Backend** (Node.js/Express) - REST API server
3. **ML Service** (FastAPI + scikit-learn) - AI/ML microservice

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
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_management
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

# Start backend server
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
API_HOST=0.0.0.0
API_PORT=8000
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Backend API Reference

### Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// Login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};

// Get authenticated user
const getCurrentUser = async (token) => {
  const response = await fetch('http://localhost:5000/api/auth/me', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// Create user
const createUser = async (token, userData) => {
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// Update user
const updateUser = async (token, userId, updates) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (token, userId) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management

```javascript
// Get all tasks
const getTasks = async (token, filters = {}) => {
  const queryParams = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tasks?${queryParams}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// Create task
const createTask = async (token, taskData) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority || 'medium',
      status: taskData.status || 'todo',
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (token, taskId, newStatus) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// Track time on task
const logTaskTime = async (token, taskId, timeData) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      duration: timeData.duration, // in seconds
      note: timeData.note
    })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (token, ticketData) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority || 'medium'
    })
  });
  return response.json();
};

// Get user's tickets
const getUserTickets = async (token) => {
  const response = await fetch('http://localhost:5000/api/tickets/my-tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// Update ticket (Admin)
const updateTicket = async (token, ticketId, updates) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API Reference

### Risk Detection

```javascript
// Analyze user risk based on behavior patterns
const analyzeUserRisk = async (userId, activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: activityData.loginFrequency,
      failed_login_attempts: activityData.failedLogins,
      task_completion_rate: activityData.completionRate,
      avg_task_duration: activityData.avgTaskDuration,
      unusual_access_times: activityData.unusualAccessTimes
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.75, risk_level: "high", factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      activity_pattern: behaviorData.activityPattern,
      access_locations: behaviorData.locations,
      work_hours: behaviorData.workHours,
      data_access_volume: behaviorData.dataVolume
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.82, details: {...} }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze employee burnout risk
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      user_id: userId,
      active_tasks: workloadData.activeTasks,
      overdue_tasks: workloadData.overdueTasks,
      avg_daily_hours: workloadData.avgDailyHours,
      weekend_work_frequency: workloadData.weekendWork,
      task_completion_trend: workloadData.completionTrend
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: "high", score: 0.78, recommendations: [...] }
  return result;
};
```

### Ticket Classification

```javascript
// Classify support ticket using AI
const classifyTicket = async (ticketData) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description
    })
  });
  const result = await response.json();
  // Returns: { category: "technical", priority: "high", suggested_assignee: "tech-team" }
  return result;
};
```

### Predictive Insights

```javascript
// Predict project delay risk
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      avg_completion_time: projectData.avgCompletionTime,
      deadline: projectData.deadline
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.65, estimated_completion: "2026-05-15", risk_factors: [...] }
  return result;
};
```

## React Frontend Patterns

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchCurrentUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchCurrentUser = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    
    if (data.token) {
      setToken(data.token);
      setUser(data.user);
      localStorage.setItem('token', data.token);
      return true;
    }
    return false;
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.tasks.filter(t => t.status === 'todo'),
      inProgress: data.tasks.filter(t => t.status === 'in-progress'),
      done: data.tasks.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = async (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    
    fetchTasks();
  };

  const renderColumn = (columnName, columnTasks, status) => (
    <div 
      className="kanban-column"
      onDragOver={(e) => e.preventDefault()}
      onDrop={(e) => onDrop(e, status)}
    >
      <h3>{columnName}</h3>
      {columnTasks.map(task => (
        <div
          key={task._id}
          className="task-card"
          draggable
          onDragStart={(e) => onDragStart(e, task._id)}
        >
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('To Do', tasks.todo, 'todo')}
      {renderColumn('In Progress', tasks.inProgress, 'in-progress')}
      {renderColumn('Done', tasks.done, 'done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);

  const fetchAIAnalytics = async () => {
    // Fetch user activity data first
    const activityResponse = await fetch(`http://localhost:5000/api/users/${userId}/activity`);
    const activityData = await activityResponse.json();

    // Risk Detection
    const riskResponse = await fetch('http://localhost:8000/api/ml/risk-detection', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: userId,
        login_frequency: activityData.loginFrequency,
        failed_login_attempts: activityData.failedLogins,
        task_completion_rate: activityData.completionRate
      })
    });
    const riskData = await riskResponse.json();

    // Burnout Detection
    const burnoutResponse = await fetch('http://localhost:8000/api/ml/burnout-detection', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        user_id: userId,
        active_tasks: activityData.activeTasks,
        avg_daily_hours: activityData.avgDailyHours
      })
    });
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskScore: riskData,
      burnoutRisk: burnoutData,
      anomalies: []
    });
  };

  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card">
        <h3>Risk Score</h3>
        <div className={`risk-indicator risk-${analytics.riskScore?.risk_level}`}>
          {analytics.riskScore?.risk_score?.toFixed(2) || 'N/A'}
        </div>
        <p>Level: {analytics.riskScore?.risk_level || 'Unknown'}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Risk</h3>
        <div className={`burnout-indicator burnout-${analytics.burnoutRisk?.burnout_risk}`}>
          {analytics.burnoutRisk?.score?.toFixed(2) || 'N/A'}
        </div>
        {analytics.burnoutRisk?.recommendations?.map((rec, idx) => (
          <p key={idx}>{rec}</p>
        ))}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration

### Backend Configuration (backend/config/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoURI: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceURL: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  maxLoginAttempts: 5,
  lockoutDuration: 15 * 60 * 1000, // 15 minutes
  roles: {
    ADMIN: 'admin',
    USER: 'user',
    MANAGER: 'manager'
  }
};
```

### ML Service Configuration (ml-service/config.py)

```python
# ml-service/config.py
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    API_HOST: str = os.getenv('API_HOST', '0.0.0.0')
    API_PORT: int = int(os.getenv('API_PORT', 8000))
    MODEL_PATH: str = os.getenv('MODEL_PATH', './models')
    LOG_LEVEL: str = os.getenv('LOG_LEVEL', 'info')
    
    # ML Model Parameters
    RISK_THRESHOLD: float = 0.7
    BURNOUT_THRESHOLD: float = 0.6
    ANOMALY_THRESHOLD: float = 0.8
    
    class Config:
        env_file = '.env'

settings = Settings()
```

## ML Model Implementation

### Risk Detection Model (ml-service/models/risk_detector.py)

```python
# ml-service/models/risk_detector.py
from sklearn.ensemble import RandomForestClassifier
import numpy as np
import joblib
import os

class RiskDetector:
    def __init__(self, model_path='./models'):
        self.model_path = model_path
        self.model = None
        self.load_or_initialize()
    
    def load_or_initialize(self):
        model_file = os.path.join(self.model_path, 'risk_model.pkl')
        if os.path.exists(model_file):
            self.model = joblib.load(model_file)
        else:
            self.model = RandomForestClassifier(
                n_estimators=100,
                max_depth=10,
                random_state=42
            )
    
    def predict_risk(self, features):
        """
        features: dict with keys:
        - login_frequency
        - failed_login_attempts
        - task_completion_rate
        - avg_task_duration
        - unusual_access_times
        """
        feature_vector = np.array([[
            features.get('login_frequency', 0),
            features.get('failed_login_attempts', 0),
            features.get('task_completion_rate', 0),
            features.get('avg_task_duration', 0),
            features.get('unusual_access_times', 0)
        ]])
        
        if hasattr(self.model, 'predict_proba'):
            risk_score = self.model.predict_proba(feature_vector)[0][1]
        else:
            # Fallback: rule-based scoring
            risk_score = self._calculate_rule_based_risk(features)
        
        risk_level = self._get_risk_level(risk_score)
        factors = self._identify_risk_factors(features)
        
        return {
            'risk_score': float(risk_score),
            'risk_level': risk_level,
            'factors': factors
        }
    
    def _calculate_rule_based_risk(self, features):
        score = 0.0
        
        if features.get('failed_login_attempts', 0) > 3:
            score += 0.3
        
        if features.get('task_completion_rate', 1.0) < 0.5:
            score += 0.2
        
        if features.get('unusual_access_times', 0) > 5:
            score += 0.25
        
        if features.get('login_frequency', 0) < 2:
            score += 0.15
        
        return min(score, 1.0)
    
    def _get_risk_level(self, score):
        if score >= 0.7:
            return 'high'
        elif score >= 0.4:
            return 'medium'
        else:
            return 'low'
    
    def _identify_risk_factors(self, features):
        factors = []
        
        if features.get('failed_login_attempts', 0) > 3:
            factors.append('High failed login attempts')
        
        if features.get('task_completion_rate', 1.0) < 0.5:
            factors.append('Low task completion rate')
        
        if features.get('unusual_access_times', 0) > 5:
            factors.append('Unusual access patterns')
        
        return factors
    
    def save_model(self):
        os.makedirs(self.model_path, exist_ok=True)
        model_file = os.path.join(self.model_path, 'risk_model.pkl')
        joblib.dump(self.model, model_file)
```

### FastAPI ML Endpoints (ml-service/main.py)

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from models.risk_detector import RiskDetector
from models.burnout_detector import BurnoutDetector
from models.anomaly_detector import AnomalyDetector
from models.ticket_classifier import TicketClassifier
import uvicorn

app = FastAPI(title="Enterprise User Management ML Service")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize models
risk_detector = RiskDetector()
burnout_detector = BurnoutDetector()
anomaly_detector = AnomalyDetector()
ticket_classifier = TicketClassifier()

class RiskRequest(BaseModel):
    user_id: str
    login_frequency: int
    failed_login_attempts: int
    task_completion_rate: float
    avg_task_duration: float
    unusual_access_times: int

class BurnoutRequest(BaseModel):
    user_id: str
    active_tasks: int
    overdue_tasks: int
    avg_daily_hours: float
    weekend_work_frequency: int
    task_completion_trend: float

class TicketRequest(BaseModel):
    subject: str
    description: str

@app.post("/api/ml/risk-detection")
async def detect_risk(request: RiskRequest):
    try:
        result = risk_detector.predict_risk({
            'login_frequency': request.login_frequency,
            'failed_login_attempts': request.failed_login_attempts,
            'task_completion_rate': request.task_completion_rate,
            'avg_task_duration': request.avg_task_duration,
            'unusual_access_times': request.unusual_access_times
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/burnout-detection")
async def detect_burnout(request: BurnoutRequest):
    try:
        result = burnout_detector.predict_burnout({
            'active_tasks': request.active_tasks,
            'overdue_tasks': request.overdue_tasks,
            'avg_daily_hours': request.avg_daily_hours,
            'weekend_work_frequency': request.weekend_work_frequency,
            'task_completion_trend': request.task_completion_trend
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/ml/classify-ticket")
async def classify_ticket(request: TicketRequest):
    try:
        result = ticket_classifier.classify({
            'subject': request.subject,
            'description': request.description
        })
        return result
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "ml-service"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Common Patterns

### Protected Route Component

```javascript
// src/components/ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) {
    return <div>Loading...</div>;
  }

  if (!user) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Time Tracking Hook

```javascript
// src/hooks/useTimeTracker.js
import { useState, useEffect, useRef } from 'react';

export const useTimeTracker = (taskId, token) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const intervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      intervalRef.current = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    }

    return () => {
      if (intervalRef.current) {
        clearInterval(intervalRef.current);
      }
    };
  }, [isRunning]);

  const start = () => setIsRunning(true);
  
  const stop = async () => {
    setIsRunning(false);
    
    if (seconds > 0) {
      await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
        method: 'POST',
        headers: {
          'Authorization': `
