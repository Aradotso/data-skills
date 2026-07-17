---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with task tracking"
  - "build admin dashboard with AI insights"
  - "integrate ML service for user behavior analysis"
  - "deploy user management system with kanban board"
  - "configure JWT authentication for user management"
  - "implement AI ticket classification system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support ticketing with machine learning-powered insights. It provides:

- **User Management**: Role-based access control (Admin/User) with JWT authentication
- **Task Tracking**: Kanban board workflow with time tracking
- **Support Tickets**: AI-classified and auto-routed ticket system
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, and project delay forecasting
- **Dashboard Analytics**: Real-time metrics for organizational performance

The system consists of three main components:
- **Frontend**: React.js SPA for user/admin interfaces
- **Backend**: Node.js REST API with MongoDB
- **ML Service**: FastAPI microservice with scikit-learn and River for online learning

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+ required
python --version  # Python 3.8+ required
mongod --version  # MongoDB 4.4+ required
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
MONGO_URI=mongodb://localhost:27017/enterprise_users
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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
MODEL_PATH=./models
LOG_LEVEL=info
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
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

The application will be available at:
- Frontend: http://localhost:3000
- Backend API: http://localhost:5000
- ML Service: http://localhost:8000

## Architecture

### Backend API Endpoints

#### Authentication
```javascript
// POST /api/auth/register
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "securepassword",
  "role": "user"
}

// POST /api/auth/login
{
  "email": "john@example.com",
  "password": "securepassword"
}
// Returns: { token: "jwt_token", user: {...} }
```

#### User Management (Admin Only)
```javascript
// GET /api/users - List all users
// GET /api/users/:id - Get user by ID
// PUT /api/users/:id - Update user
// DELETE /api/users/:id - Delete user

// Example: Update user role
fetch('http://localhost:5000/api/users/123', {
  method: 'PUT',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    role: 'admin',
    status: 'active'
  })
});
```

#### Task Management
```javascript
// POST /api/tasks - Create task
{
  "title": "Implement user dashboard",
  "description": "Build React dashboard with metrics",
  "assignedTo": "user_id",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}

// PUT /api/tasks/:id - Update task status
{
  "status": "in-progress",
  "timeSpent": 120  // minutes
}

// GET /api/tasks/user/:userId - Get user tasks
// GET /api/tasks/analytics - Get task analytics
```

#### Support Tickets
```javascript
// POST /api/tickets - Create ticket
{
  "title": "Cannot access dashboard",
  "description": "Getting 403 error when accessing /dashboard",
  "priority": "medium",
  "category": "technical"
}

// PUT /api/tickets/:id - Update ticket
{
  "status": "in-progress",
  "assignedTo": "admin_id",
  "aiClassification": "authentication_issue"
}
```

### ML Service API

#### Risk Detection
```python
# POST /api/ml/risk-detection
# Analyzes user behavior for security risks

import requests

response = requests.post(
    'http://localhost:8000/api/ml/risk-detection',
    json={
        "user_id": "user123",
        "login_frequency": 15,
        "failed_logins": 3,
        "unusual_hours": 2,
        "data_access_volume": 1500
    }
)

# Returns: { "risk_score": 0.75, "risk_level": "high", "factors": [...] }
```

#### Ticket Classification
```python
# POST /api/ml/classify-ticket
# Auto-classifies and routes support tickets

response = requests.post(
    'http://localhost:8000/api/ml/classify-ticket',
    json={
        "title": "Password reset not working",
        "description": "I clicked forgot password but didn't receive email",
        "user_history": []
    }
)

# Returns: { "category": "authentication", "priority": "high", "suggested_assignee": "admin_id" }
```

#### Burnout Detection
```python
# POST /api/ml/burnout-analysis
# Analyzes workload and task patterns for burnout risk

response = requests.post(
    'http://localhost:8000/api/ml/burnout-analysis',
    json={
        "user_id": "user123",
        "tasks_completed": 45,
        "avg_task_time": 180,
        "overtime_hours": 15,
        "task_complexity": 7.5,
        "days_since_break": 21
    }
)

# Returns: { "burnout_score": 0.82, "risk_level": "high", "recommendations": [...] }
```

#### Predictive Project Insights
```python
# POST /api/ml/project-prediction
# Predicts project delays and completion times

response = requests.post(
    'http://localhost:8000/api/ml/project-prediction',
    json={
        "project_id": "proj123",
        "total_tasks": 50,
        "completed_tasks": 20,
        "avg_completion_rate": 0.6,
        "team_size": 5,
        "deadline_days": 30
    }
)

# Returns: { "estimated_completion": "2026-06-15", "delay_risk": 0.65, "recommendations": [...] }
```

## Frontend Integration Patterns

### Authentication Flow

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
      loadUser();
    } else {
      setLoading(false);
    }
  }, []);

  const loadUser = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(res.data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const res = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUser(res.data.user);
    return res.data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, loading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    try {
      const res = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/user/${userId}`
      );
      
      const grouped = {
        todo: res.data.filter(t => t.status === 'todo'),
        inProgress: res.data.filter(t => t.status === 'in-progress'),
        done: res.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.put(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks();
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const onDragStart = (e, taskId, currentStatus) => {
    e.dataTransfer.setData('taskId', taskId);
    e.dataTransfer.setData('currentStatus', currentStatus);
  };

  const onDrop = (e, newStatus) => {
    const taskId = e.dataTransfer.getData('taskId');
    updateTaskStatus(taskId, newStatus);
  };

  const onDragOver = (e) => {
    e.preventDefault();
  };

  if (loading) return <div>Loading tasks...</div>;

  return (
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
// src/components/AdminAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [aiInsights, setAiInsights] = useState(null);

  useEffect(() => {
    fetchAnalytics();
    fetchAIInsights();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const res = await axios.get(`${process.env.REACT_APP_API_URL}/analytics/overview`);
      setAnalytics(res.data);
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  const fetchAIInsights = async () => {
    try {
      // Risk detection for all users
      const riskRes = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-overview`
      );
      
      // Burnout analysis
      const burnoutRes = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-overview`
      );
      
      // Project predictions
      const projectRes = await axios.get(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/project-risks`
      );
      
      setAiInsights({
        risks: riskRes.data,
        burnout: burnoutRes.data,
        projects: projectRes.data
      });
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    }
  };

  if (!analytics || !aiInsights) return <div>Loading analytics...</div>;

  return (
    <div className="analytics-dashboard">
      <div className="metrics-grid">
        <div className="metric-card">
          <h3>Total Users</h3>
          <p className="metric-value">{analytics.totalUsers}</p>
        </div>
        <div className="metric-card">
          <h3>Active Tasks</h3>
          <p className="metric-value">{analytics.activeTasks}</p>
        </div>
        <div className="metric-card">
          <h3>Open Tickets</h3>
          <p className="metric-value">{analytics.openTickets}</p>
        </div>
      </div>

      <div className="ai-insights">
        <h2>AI-Powered Insights</h2>
        
        <div className="insight-section">
          <h3>High-Risk Users</h3>
          {aiInsights.risks.highRisk.map(user => (
            <div key={user.id} className="alert-card">
              <p><strong>{user.name}</strong> - Risk Score: {user.score}</p>
              <p>{user.reason}</p>
            </div>
          ))}
        </div>

        <div className="insight-section">
          <h3>Burnout Risk</h3>
          {aiInsights.burnout.atRisk.map(user => (
            <div key={user.id} className="warning-card">
              <p><strong>{user.name}</strong> - Burnout Score: {user.score}</p>
              <p>Recommendations: {user.recommendations.join(', ')}</p>
            </div>
          ))}
        </div>

        <div className="insight-section">
          <h3>Project Delays</h3>
          {aiInsights.projects.delayed.map(project => (
            <div key={project.id} className="info-card">
              <p><strong>{project.name}</strong> - Delay Risk: {project.delayRisk}%</p>
              <p>Estimated Completion: {project.estimatedCompletion}</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

### Time Tracking Component

```javascript
// src/components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const [totalTime, setTotalTime] = useState(0);

  useEffect(() => {
    let interval = null;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else if (!isRunning && seconds !== 0) {
      clearInterval(interval);
    }
    return () => clearInterval(interval);
  }, [isRunning, seconds]);

  const startTimer = () => {
    setIsRunning(true);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    
    // Save time to backend
    try {
      await axios.post(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/time`, {
        timeSpent: Math.floor(seconds / 60) // Convert to minutes
      });
      
      setTotalTime(totalTime + seconds);
      setSeconds(0);
    } catch (error) {
      console.error('Failed to save time:', error);
    }
  };

  const formatTime = (secs) => {
    const hours = Math.floor(secs / 3600);
    const minutes = Math.floor((secs % 3600) / 60);
    const remainingSeconds = secs % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${remainingSeconds.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">
        <h3>{formatTime(seconds)}</h3>
        <p>Total: {formatTime(totalTime)}</p>
      </div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer} className="btn-start">Start</button>
        ) : (
          <button onClick={stopTimer} className="btn-stop">Stop</button>
        )}
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise_users
JWT_SECRET=${JWT_SECRET}  # Generate with: openssl rand -base64 32
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
CORS_ORIGIN=http://localhost:3000

# Optional: Email service for notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}
```

**ML Service (.env)**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise_users
MODEL_PATH=./models
LOG_LEVEL=info
RETRAIN_INTERVAL=86400  # Retrain models daily (seconds)
RISK_THRESHOLD=0.7
BURNOUT_THRESHOLD=0.75
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.jsx
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;

  if (!user) return <Navigate to="/login" />;

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter as Router, Route, Routes } from 'react-router-dom';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly={true}>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </Router>
  );
}
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    // Poll for notifications every 30 seconds
    const fetchNotifications = async () => {
      try {
        const res = await axios.get(
          `${process.env.REACT_APP_API_URL}/notifications/${userId}`
        );
        setNotifications(res.data);
      } catch (error) {
        console.error('Failed to fetch notifications:', error);
      }
    };

    fetchNotifications();
    const interval = setInterval(fetchNotifications, 30000);

    return () => clearInterval(interval);
  }, [userId]);

  const markAsRead = async (notificationId) => {
    try {
      await axios.put(
        `${process.env.REACT_APP_API_URL}/notifications/${notificationId}/read`
      );
      setNotifications(prev => 
        prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
      );
    } catch (error) {
      console.error('Failed to mark notification as read:', error);
    }
  };

  return { notifications, markAsRead };
};
```

### Audit Logging

```javascript
// backend/middleware/auditLog.js
const AuditLog = require('../models/AuditLog');

const auditLog = (action) => async (req, res, next) => {
  try {
    await AuditLog.create({
      user: req.user.id,
      action: action,
      resource: req.originalUrl,
      method: req.method,
      ip: req.ip,
      userAgent: req.get('User-Agent'),
      timestamp: new Date()
    });
  } catch (error) {
    console.error('Audit log failed:', error);
  }
  next();
};

// Usage in routes
router.delete('/users/:id', 
  protect, 
  authorize('admin'), 
  auditLog('DELETE_USER'),
  deleteUser
);
```

## Troubleshooting

### Issue: JWT Token Expired

```javascript
// src/utils/axiosInterceptor.js
import axios from 'axios';

axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;

    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh token logic
        const refreshToken = localStorage.getItem('refreshToken');
        const res = await axios.post('/api/auth/refresh', { refreshToken });
        
        localStorage.setItem('token', res.data.token);
        axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
        
        return axios(originalRequest);
      } catch (refreshError) {
        // Redirect to login
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

### Issue: ML Service Connection Failed

Check if ML service is running and accessible:

```bash
# Test ML service health
curl http://localhost:8000/health

# Check logs
cd ml-service
tail -f logs/ml_service.log

# Restart ML service
uvicorn main:app --reload --port 8000 --log-level debug
```

### Issue: MongoDB Connection Error

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });

    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### Issue: CORS Errors

```javascript
// backend/server.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));

// Handle preflight requests
app.options('*', cors(corsOptions));
```

### Issue: ML Model Not Loading

```python
# ml-service/utils/model_loader.py
import os
import pickle
import logging

logger = logging.getLogger(__name__)

def load_model(model_name):
    model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
    
    if not os.path.exists(model_path):
        logger.warning(f'Model {model_name} not found. Training new model...')
        # Train and save new model
        model = train_new_model(model_name)
        save_model(model, model_path)
        return model
    
    try:
        with open(model_path, 'rb') as f:
            model = pickle.load(f)
        logger.info(f'Model {model_name} loaded successfully')
        return model
    except Exception as e:
        logger.error(f'Failed to load model {model_name}: {str(e)}')
        raise
```

## Testing

### Backend API Tests

```javascript
// backend/tests/auth.test.js
const request = require('supertest');
const app = require('../app');
const mongoose = require('mongoose');

describe('Authentication', () => {
  beforeAll(async () => {
    await mongoose.connect(process.env.MONGO_URI_TEST);
  });

  afterAll(async () => {
    await mongoose.connection.close();
  });

  test('should register new user', async () => {
    const res = await request(app)
      .post('/api/auth/register')
      .send({
        name: 'Test User',
        email: 'test@example.com',
        password: 'password123',
        role: 'user'
      });

    expect(res.statusCode).toBe(201);
    expect(res.body).toHaveProperty('token');
  });

  test('should login existing user', async () => {
    const res = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123'
      });

    expect(res.statusCode).toBe(200);
    expect(res.body).toHaveProperty('token');
  });
});
```

### Frontend Component Tests

```javascript
// frontend/src/components/__tests__/KanbanBoard.test.js
import { render, screen, waitFor } from '@testing-library/react';
import KanbanBoard from '../KanbanBoard';
import axios from 'axios';

jest.mock('axios');

describe('KanbanBoard', () => {
  test('renders task columns', async () => {
    axios.get.mockResolvedValue({
      data: [
        { _id: '1', title: 'Task 1', status: 'todo' },
        { _id: '2', title: 'Task 2', status: 'in-progress' }
      ]
    });

    render(<KanbanBoard userId="user123" />);

    await waitFor(() => {
      expect(screen.getByText('TODO')).toBeInTheDocument();
      expect(screen.getByText('IN PROGRESS')).toBeInTheDocument();
      expect(screen.getByText('DONE')).toBeInTheDocument();
    });
  });
});
```

This skill provides comprehensive guidance for working with the Enterprise User Management System with AI Analytics, covering installation, API usage, frontend patterns, ML integration, and troubleshooting common issues.
