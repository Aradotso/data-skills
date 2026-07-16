---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered risk detection, anomaly detection, and predictive analytics for enterprise workflows
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management dashboard with ML"
  - "add risk detection to user management"
  - "build task tracking with burnout detection"
  - "integrate AI ticket classification system"
  - "deploy user management with predictive insights"
  - "configure enterprise system with AI analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with ML-powered insights. It provides role-based access control, Kanban task boards, ticket management, and AI features including risk prediction, anomaly detection, burnout analysis, and predictive project insights. Built with React frontend, Node.js backend, FastAPI ML service, and MongoDB database.

## Installation

### Prerequisites

```bash
# Node.js 16+ and Python 3.8+ required
node --version
python --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend (.env in backend/)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env in ml-service/)**
```bash
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=INFO
```

**Frontend (.env in frontend/)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the System

### Start Backend Server

```bash
cd backend
npm start
# Server runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload
# ML service runs at http://localhost:8000
```

### Start Frontend

```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication

```javascript
// Login endpoint
POST /api/auth/login
Body: {
  "email": "user@example.com",
  "password": "password123"
}

// Register new user
POST /api/auth/register
Body: {
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepass",
  "role": "user"
}
```

### User Management (Admin)

```javascript
// Get all users
GET /api/users
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Create user
POST /api/users
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "name": "Jane Smith",
  "email": "jane@company.com",
  "role": "user",
  "department": "Engineering"
}

// Update user
PUT /api/users/:userId
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "role": "manager",
  "status": "active"
}

// Delete user
DELETE /api/users/:userId
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
```

### Task Management

```javascript
// Get user tasks
GET /api/tasks
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Create task
POST /api/tasks
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "title": "Implement feature X",
  "description": "Add new analytics dashboard",
  "assignedTo": "userId123",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// Update task status
PATCH /api/tasks/:taskId
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "status": "in_progress"
}

// Track time on task
POST /api/tasks/:taskId/time
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "duration": 3600 // seconds
}
```

### Support Tickets

```javascript
// Create ticket
POST /api/tickets
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "title": "Login issue",
  "description": "Cannot access dashboard",
  "priority": "medium",
  "category": "technical"
}

// Get tickets (admin)
GET /api/tickets/all
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }

// Assign ticket (admin)
PATCH /api/tickets/:ticketId/assign
Headers: { "Authorization": "Bearer <JWT_TOKEN>" }
Body: {
  "assignedTo": "supportUserId"
}
```

## ML Service API Reference

### Risk Prediction

```javascript
// Predict user risk score
POST /ml/predict-risk
Body: {
  "userId": "user123",
  "loginAttempts": 5,
  "failedLogins": 2,
  "lastLoginTime": "2026-04-15T10:30:00Z",
  "taskCompletionRate": 0.75,
  "averageResponseTime": 120
}

Response: {
  "riskScore": 0.35,
  "riskLevel": "medium",
  "factors": ["increased_failed_logins", "response_time_delay"]
}
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
POST /ml/detect-anomaly
Body: {
  "userId": "user123",
  "loginTime": "2026-04-15T03:00:00Z",
  "ipAddress": "192.168.1.100",
  "loginLocation": "New York",
  "deviceType": "mobile"
}

Response: {
  "isAnomaly": true,
  "anomalyScore": 0.89,
  "reasons": ["unusual_login_time", "new_device"]
}
```

### Burnout Detection

```javascript
// Analyze user burnout risk
POST /ml/burnout-analysis
Body: {
  "userId": "user123",
  "tasksAssigned": 25,
  "tasksCompleted": 18,
  "averageWorkHours": 11.5,
  "overtimeHours": 20,
  "lastBreakDays": 45
}

Response: {
  "burnoutRisk": "high",
  "burnoutScore": 0.82,
  "recommendations": [
    "reduce_workload",
    "schedule_break",
    "redistribute_tasks"
  ]
}
```

### Predictive Project Insights

```javascript
// Predict project completion
POST /ml/project-prediction
Body: {
  "projectId": "proj123",
  "totalTasks": 50,
  "completedTasks": 30,
  "daysElapsed": 20,
  "estimatedDays": 30,
  "teamSize": 5,
  "averageVelocity": 2.5
}

Response: {
  "predictedDelay": 5,
  "completionProbability": 0.65,
  "recommendedActions": ["add_resources", "adjust_scope"]
}
```

### AI Ticket Classification

```javascript
// Classify support ticket
POST /ml/classify-ticket
Body: {
  "title": "Cannot login to system",
  "description": "Getting error 500 when trying to authenticate",
  "userRole": "employee"
}

Response: {
  "category": "technical",
  "priority": "high",
  "suggestedAssignee": "tech_support_team",
  "estimatedResolutionTime": 2
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

### Task Management Component

```javascript
// src/components/TaskBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TaskBoard = () => {
  const [tasks, setTasks] = useState({
    todo: [],
    inProgress: [],
    done: []
  });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/tasks`);
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in_progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${API_URL}/api/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  const renderColumn = (title, status, taskList) => (
    <div className="task-column">
      <h3>{title}</h3>
      {taskList.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>
            {task.priority}
          </span>
          <select 
            value={status}
            onChange={(e) => updateTaskStatus(task._id, e.target.value)}
          >
            <option value="todo">To Do</option>
            <option value="in_progress">In Progress</option>
            <option value="done">Done</option>
          </select>
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('To Do', 'todo', tasks.todo)}
      {renderColumn('In Progress', 'in_progress', tasks.inProgress)}
      {renderColumn('Done', 'done', tasks.done)}
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

const AIAnalytics = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, [userId]);

  const fetchAnalytics = async () => {
    try {
      // Get user data
      const userResponse = await axios.get(`${API_URL}/api/users/${userId}`);
      const userData = userResponse.data;

      // Risk prediction
      const riskResponse = await axios.post(`${ML_API_URL}/ml/predict-risk`, {
        userId: userId,
        loginAttempts: userData.loginAttempts || 0,
        failedLogins: userData.failedLogins || 0,
        taskCompletionRate: userData.taskCompletionRate || 0,
        averageResponseTime: userData.avgResponseTime || 0
      });

      // Burnout analysis
      const burnoutResponse = await axios.post(`${ML_API_URL}/ml/burnout-analysis`, {
        userId: userId,
        tasksAssigned: userData.tasksAssigned || 0,
        tasksCompleted: userData.tasksCompleted || 0,
        averageWorkHours: userData.avgWorkHours || 8,
        overtimeHours: userData.overtimeHours || 0,
        lastBreakDays: userData.daysSinceBreak || 0
      });

      setAnalytics({
        risk: riskResponse.data,
        burnout: burnoutResponse.data,
        anomalies: []
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="ai-analytics">
      <h2>AI-Powered Insights</h2>
      
      {analytics.risk && (
        <div className={`risk-card risk-${analytics.risk.riskLevel}`}>
          <h3>Risk Score: {(analytics.risk.riskScore * 100).toFixed(1)}%</h3>
          <p>Level: {analytics.risk.riskLevel}</p>
          <ul>
            {analytics.risk.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.burnout && (
        <div className={`burnout-card burnout-${analytics.burnout.burnoutRisk}`}>
          <h3>Burnout Risk: {analytics.burnout.burnoutRisk}</h3>
          <p>Score: {(analytics.burnout.burnoutScore * 100).toFixed(1)}%</p>
          <h4>Recommendations:</h4>
          <ul>
            {analytics.burnout.recommendations.map((rec, idx) => (
              <li key={idx}>{rec.replace(/_/g, ' ')}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalytics;
```

### Ticket Classification Helper

```javascript
// src/utils/ticketHelper.js
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;
const API_URL = process.env.REACT_APP_API_URL;

export const createSmartTicket = async (ticketData) => {
  try {
    // Classify ticket using ML
    const classificationResponse = await axios.post(
      `${ML_API_URL}/ml/classify-ticket`,
      {
        title: ticketData.title,
        description: ticketData.description,
        userRole: ticketData.userRole
      }
    );

    const classification = classificationResponse.data;

    // Create ticket with AI suggestions
    const ticketResponse = await axios.post(`${API_URL}/api/tickets`, {
      ...ticketData,
      category: classification.category,
      priority: classification.priority,
      suggestedAssignee: classification.suggestedAssignee,
      estimatedResolution: classification.estimatedResolutionTime
    });

    return ticketResponse.data;
  } catch (error) {
    console.error('Error creating smart ticket:', error);
    throw error;
  }
};

export const getTicketInsights = async (ticketId) => {
  try {
    const response = await axios.get(`${API_URL}/api/tickets/${ticketId}/insights`);
    return response.data;
  } catch (error) {
    console.error('Error fetching ticket insights:', error);
    throw error;
  }
};
```

## Common Patterns

### Protected Route Wrapper

```javascript
// src/components/ProtectedRoute.jsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, loading } = useAuth();

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

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const TimeTracker = ({ taskId }) => {
  const [isTracking, setIsTracking] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [startTime, setStartTime] = useState(null);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const startTracking = () => {
    setStartTime(Date.now());
    setIsTracking(true);
  };

  const stopTracking = async () => {
    setIsTracking(false);
    const duration = Math.floor(elapsedTime / 1000);
    
    try {
      await axios.post(`${API_URL}/api/tasks/${taskId}/time`, {
        duration: duration
      });
      setElapsedTime(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
  };

  const formatTime = (ms) => {
    const seconds = Math.floor(ms / 1000) % 60;
    const minutes = Math.floor(ms / 60000) % 60;
    const hours = Math.floor(ms / 3600000);
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(elapsedTime)}</div>
      {!isTracking ? (
        <button onClick={startTracking}>Start</button>
      ) : (
        <button onClick={stopTracking}>Stop</button>
      )}
    </div>
  );
};

export default TimeTracker;
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection string in backend/.env
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
```

### JWT Token Expiration

```javascript
// Add token refresh logic in frontend
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Clear token and redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Not Responding

```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Verify Python dependencies
pip list | grep -E "fastapi|scikit-learn|river"

# Reinstall if needed
pip install --upgrade fastapi scikit-learn river uvicorn
```

### CORS Issues

```javascript
// In backend/server.js or app.js
const cors = require('cors');
app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Optimization

```javascript
// Add pagination to user list
GET /api/users?page=1&limit=20

// Backend implementation
const page = parseInt(req.query.page) || 1;
const limit = parseInt(req.query.limit) || 20;
const skip = (page - 1) * limit;

const users = await User.find()
  .skip(skip)
  .limit(limit)
  .select('-password');
```

### Model Retraining

```bash
# Retrain ML models with new data
cd ml-service
python scripts/train_models.py --data ./data/user_behavior.csv

# Update model path in environment
MODEL_PATH=./models/latest
```

This skill provides comprehensive guidance for deploying and using the Enterprise User Management System with AI Analytics in production environments.
