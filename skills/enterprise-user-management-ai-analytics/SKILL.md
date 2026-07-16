---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, and task automation using React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement task tracking with burnout detection"
  - "create admin dashboard with risk prediction"
  - "build ticket management with AI classification"
  - "configure user roles and authentication with JWT"
  - "deploy user management system with ML service"
  - "add anomaly detection to enterprise application"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A comprehensive full-stack application for managing enterprise users, tasks, and support tickets with integrated AI analytics including risk detection, anomaly detection, burnout analysis, and predictive insights.

## What It Does

This system provides:
- **User Management**: Role-based access control, authentication via JWT
- **Task Management**: Kanban boards, time tracking, assignment workflows
- **Support Tickets**: AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Dashboards**: Separate admin and user interfaces with real-time insights

## Architecture

The project consists of three main components:
1. **Frontend** (React.js) - User and admin dashboards
2. **Backend** (Node.js/Express) - REST API server
3. **ML Service** (FastAPI + scikit-learn) - AI/ML endpoints

## Installation

### Prerequisites

```bash
# Required software
node >= 14.x
npm >= 6.x
python >= 3.8
pip >= 20.x
mongodb >= 4.x
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
```

Create `.env` file in `backend/` directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# or for development with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/` directory:

```env
MONGO_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# or
python -m uvicorn main:app --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in `frontend/` directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
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
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

### Ticket Management Endpoints

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets
const getTickets = async (token, filters = {}) => {
  const params = new URLSearchParams(filters);
  const response = await fetch(`http://localhost:5000/api/tickets?${params}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### AI Analytics Endpoints

```javascript
// POST /api/ml/risk-prediction - Predict user risk
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 45,
        task_completion_rate: 0.75,
        avg_response_time: 120,
        failed_login_attempts: 2
      }
    })
  });
  return response.json();
  // Returns: { risk_score: 0.35, risk_level: 'low', factors: [...] }
};

// POST /api/ml/anomaly-detection - Detect anomalous behavior
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      activity_log: {
        timestamp: new Date().toISOString(),
        action: activityData.action,
        ip_address: activityData.ip,
        device: activityData.device
      }
    })
  });
  return response.json();
  // Returns: { is_anomaly: false, anomaly_score: 0.12 }
};

// POST /api/ml/burnout-analysis - Analyze employee burnout risk
const analyzeBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-analysis', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      metrics: {
        avg_hours_per_week: 52,
        tasks_overdue: 5,
        stress_indicators: 7,
        time_off_days: 2
      }
    })
  });
  return response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
};

// POST /api/ml/ticket-classification - Auto-classify support ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/ticket-classification', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      metadata: {
        source: 'web',
        timestamp: new Date().toISOString()
      }
    })
  });
  return response.json();
  // Returns: { category: 'technical', priority: 'high', suggested_agent: 'tech_support' }
};

// POST /api/ml/project-insights - Get predictive project insights
const getProjectInsights = async (projectId) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectId,
      current_progress: 0.65,
      deadline: '2026-06-01',
      team_size: 8,
      task_completion_velocity: 12.5
    })
  });
  return response.json();
  // Returns: { delay_probability: 0.23, estimated_completion: '2026-05-28', risks: [...] }
};
```

## Frontend Components

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';

const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
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
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setToken(data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const { token, user } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${user.id}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  const Column = ({ title, status, taskList }) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {taskList.map(task => (
        <div key={task.id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
          {status !== 'done' && (
            <button onClick={() => moveTask(task.id, 
              status === 'todo' ? 'in-progress' : 'done')}>
              Move →
            </button>
          )}
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      <Column title="To Do" status="todo" taskList={tasks.todo} />
      <Column title="In Progress" status="inProgress" taskList={tasks.inProgress} />
      <Column title="Done" status="done" taskList={tasks.done} />
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';
import { useAuth } from '../hooks/useAuth';

const AIAnalyticsDashboard = () => {
  const { token, user } = useAuth();
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    // Fetch risk prediction
    const riskResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user.id, features: {} })
      }
    );
    const riskData = await riskResponse.json();

    // Fetch burnout analysis
    const burnoutResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user.id, metrics: {} })
      }
    );
    const burnoutData = await burnoutResponse.json();

    setAnalytics({
      riskScore: riskData,
      burnoutRisk: burnoutData,
      anomalies: []
    });
  };

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>
      
      {analytics.riskScore && (
        <div className="metric-card">
          <h3>Risk Assessment</h3>
          <div className={`risk-level-${analytics.riskScore.risk_level}`}>
            Score: {(analytics.riskScore.risk_score * 100).toFixed(1)}%
          </div>
          <p>Level: {analytics.riskScore.risk_level}</p>
        </div>
      )}

      {analytics.burnoutRisk && (
        <div className="metric-card">
          <h3>Burnout Risk</h3>
          <div className={`burnout-${analytics.burnoutRisk.burnout_risk}`}>
            {analytics.burnoutRisk.burnout_risk.toUpperCase()}
          </div>
          <ul>
            {analytics.burnoutRisk.recommendations?.map((rec, i) => (
              <li key={i}>{rec}</li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, requireAdmin = false }) => {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (requireAdmin && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute><UserDashboard /></ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute requireAdmin={true}><AdminDashboard /></ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Notifications

```javascript
// hooks/useNotifications.js
import { useState, useEffect } from 'react';
import { useAuth } from './useAuth';

export const useNotifications = () => {
  const { token, user } = useAuth();
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const pollNotifications = setInterval(async () => {
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}/api/notifications/${user.id}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      const data = await response.json();
      setNotifications(data);
    }, 30000); // Poll every 30 seconds

    return () => clearInterval(pollNotifications);
  }, [user, token]);

  const markAsRead = async (notificationId) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/notifications/${notificationId}/read`,
      {
        method: 'PATCH',
        headers: { 'Authorization': `Bearer ${token}` }
      }
    );
    setNotifications(prev => prev.filter(n => n.id !== notificationId));
  };

  return { notifications, markAsRead };
};
```

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, onSave }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const handleSave = () => {
    onSave(taskId, seconds);
    setSeconds(0);
    setIsRunning(false);
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={handleSave} disabled={seconds === 0}>
        Save Time
      </button>
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Backend Configuration (backend/config/database.js)

```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware (backend/middleware/auth.js)

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    MONGO_URI = os.getenv('MONGO_URI', 'mongodb://localhost:27017/enterprise-user-mgmt')
    MODEL_PATH = os.getenv('MODEL_PATH', './models')
    LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')
    
    # ML Model Parameters
    RISK_THRESHOLD = 0.7
    ANOMALY_THRESHOLD = 0.8
    BURNOUT_THRESHOLD = 0.65
    
    # Feature Configuration
    RISK_FEATURES = [
        'login_frequency',
        'task_completion_rate',
        'avg_response_time',
        'failed_login_attempts'
    ]
```

## Troubleshooting

### MongoDB Connection Issues

```javascript
// Check MongoDB is running
// Linux/Mac: sudo systemctl status mongod
// Windows: net start MongoDB

// Test connection
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected'))
  .catch(err => console.error('Connection failed:', err));
```

### JWT Token Expiration

```javascript
// Refresh token implementation
const refreshToken = async (oldToken) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${oldToken}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Token refresh failed, redirect to login
    window.location.href = '/login';
  }
};
```

### ML Service Not Responding

```bash
# Check if ML service is running
curl http://localhost:8000/health

# View logs
cd ml-service
tail -f app.log

# Restart service
pkill -f uvicorn
uvicorn main:app --reload --port 8000
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Model Training/Loading Errors

```python
# ml-service/utils/model_loader.py
import os
import pickle
from sklearn.ensemble import RandomForestClassifier

def load_or_create_model(model_path, model_name):
    full_path = os.path.join(model_path, f'{model_name}.pkl')
    
    if os.path.exists(full_path):
        with open(full_path, 'rb') as f:
            return pickle.load(f)
    else:
        # Create default model if not exists
        model = RandomForestClassifier(n_estimators=100, random_state=42)
        os.makedirs(model_path, exist_ok=True)
        with open(full_path, 'wb') as f:
            pickle.dump(model, f)
        return model
```

### Environment Variables Not Loading

```javascript
// Create .env.example files for reference
// frontend/.env.example
/*
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
*/

// backend/.env.example
/*
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your-secret-key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
*/

// Verify loading
console.log('API URL:', process.env.REACT_APP_API_URL);
```

## Production Deployment

### Build Frontend

```bash
cd frontend
npm run build
# Serve build folder with nginx or deploy to Vercel/Netlify
```

### Production Environment Variables

```bash
# Use production MongoDB cluster
MONGODB_URI=${MONGODB_ATLAS_URI}

# Secure JWT secret
JWT_SECRET=${SECURE_RANDOM_SECRET}

# Set production URLs
REACT_APP_API_URL=https://api.yourcompany.com
REACT_APP_ML_API_URL=https://ml.yourcompany.com
```

### Docker Deployment

```dockerfile
# backend/Dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:4.4
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
  
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/enterprise-user-mgmt
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - mongodb
  
  ml-service:
    build: ./ml-service
    ports:
      - "8000:8000"
    depends_on:
      - mongodb

volumes:
  mongo-data:
```
