---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user management dashboard with ML"
  - "implement ticket classification with AI"
  - "create task tracking system with burnout detection"
  - "configure user management with predictive analytics"
  - "deploy enterprise user management with AI features"
  - "setup JWT authentication for user management system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket handling with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project insights to help organizations automate workflows and improve decision-making.

**Key capabilities:**
- User authentication and role-based access control (JWT)
- Kanban-style task management with time tracking
- Support ticket system with AI-based classification
- AI analytics: risk prediction, anomaly detection, burnout analysis
- Admin dashboard with organization-wide analytics
- Real-time notifications and audit logging

## Installation

### Prerequisites

- Node.js (v14+)
- Python 3.8+
- MongoDB instance
- npm or yarn

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=24h
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

Create `.env` file for ML service:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
API_HOST=0.0.0.0
API_PORT=8000
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User interface for users and admins
2. **Backend** (Node.js) - REST API, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Backend API Usage

### Authentication

**User Registration:**

```javascript
const axios = require('axios');

async function registerUser(userData) {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    });
    return response.data;
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
}
```

**User Login:**

```javascript
async function loginUser(credentials) {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email: credentials.email,
      password: credentials.password
    });
    
    // Store JWT token
    const { token, user } = response.data;
    localStorage.setItem('authToken', token);
    localStorage.setItem('user', JSON.stringify(user));
    
    return { token, user };
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
}
```

### User Management (Admin)

```javascript
// Get all users
async function getAllUsers(token) {
  const response = await axios.get('http://localhost:5000/api/users', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Create user
async function createUser(userData, token) {
  const response = await axios.post('http://localhost:5000/api/users', userData, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Update user
async function updateUser(userId, updates, token) {
  const response = await axios.put(`http://localhost:5000/api/users/${userId}`, updates, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Delete user
async function deleteUser(userId, token) {
  const response = await axios.delete(`http://localhost:5000/api/users/${userId}`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}
```

### Task Management

```javascript
// Create task
async function createTask(taskData, token) {
  const response = await axios.post('http://localhost:5000/api/tasks', {
    title: taskData.title,
    description: taskData.description,
    assignedTo: taskData.assignedTo,
    priority: taskData.priority, // 'low', 'medium', 'high'
    status: 'todo', // 'todo', 'in_progress', 'done'
    dueDate: taskData.dueDate
  }, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Get user tasks
async function getUserTasks(token) {
  const response = await axios.get('http://localhost:5000/api/tasks/my-tasks', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Update task status
async function updateTaskStatus(taskId, status, token) {
  const response = await axios.patch(`http://localhost:5000/api/tasks/${taskId}/status`, 
    { status },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
}

// Track time on task
async function logTimeEntry(taskId, duration, token) {
  const response = await axios.post(`http://localhost:5000/api/tasks/${taskId}/time`, 
    { duration }, // duration in seconds
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
}
```

### Support Tickets

```javascript
// Create ticket
async function createTicket(ticketData, token) {
  const response = await axios.post('http://localhost:5000/api/tickets', {
    subject: ticketData.subject,
    description: ticketData.description,
    category: ticketData.category,
    priority: ticketData.priority
  }, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Get user tickets
async function getUserTickets(token) {
  const response = await axios.get('http://localhost:5000/api/tickets/my-tickets', {
    headers: { Authorization: `Bearer ${token}` }
  });
  return response.data;
}

// Update ticket status
async function updateTicket(ticketId, updates, token) {
  const response = await axios.put(`http://localhost:5000/api/tickets/${ticketId}`, 
    updates,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data;
}
```

## ML Service API Usage

### AI Ticket Classification

```python
import requests

def classify_ticket(ticket_text):
    """Classify support ticket using AI"""
    response = requests.post('http://localhost:8000/api/ml/classify-ticket', 
        json={
            'text': ticket_text,
            'subject': 'Cannot access dashboard'
        }
    )
    result = response.json()
    # Returns: {'category': 'technical', 'priority': 'high', 'confidence': 0.87}
    return result
```

```javascript
// From JavaScript/React
async function classifyTicket(ticketText, subject) {
  const response = await axios.post('http://localhost:8000/api/ml/classify-ticket', {
    text: ticketText,
    subject: subject
  });
  
  const { category, priority, confidence } = response.data;
  return { category, priority, confidence };
}
```

### Risk Prediction

```python
def predict_user_risk(user_behavior):
    """Predict risk score for user based on behavior patterns"""
    response = requests.post('http://localhost:8000/api/ml/predict-risk',
        json={
            'user_id': 'user123',
            'login_frequency': 45,
            'failed_logins': 3,
            'unusual_hours': 5,
            'data_access_volume': 150
        }
    )
    result = response.json()
    # Returns: {'risk_score': 0.65, 'risk_level': 'medium', 'factors': [...]}
    return result
```

### Anomaly Detection

```python
def detect_anomaly(user_activity):
    """Detect anomalous user behavior"""
    response = requests.post('http://localhost:8000/api/ml/detect-anomaly',
        json={
            'user_id': 'user456',
            'activity_type': 'data_download',
            'timestamp': '2024-01-15T14:30:00',
            'volume': 500,
            'location': 'unusual_geo'
        }
    )
    result = response.json()
    # Returns: {'is_anomaly': True, 'anomaly_score': 0.89, 'reason': 'Unusual data volume'}
    return result
```

### Burnout Detection

```python
def detect_burnout(workload_data):
    """Analyze user workload for burnout risk"""
    response = requests.post('http://localhost:8000/api/ml/detect-burnout',
        json={
            'user_id': 'user789',
            'hours_worked_weekly': [45, 50, 52, 48, 55],
            'tasks_completed': 25,
            'missed_deadlines': 3,
            'overtime_hours': 15
        }
    )
    result = response.json()
    # Returns: {'burnout_risk': 'high', 'score': 0.78, 'recommendations': [...]}
    return result
```

### Predictive Project Insights

```python
def predict_project_delay(project_data):
    """Predict if project will be delayed"""
    response = requests.post('http://localhost:8000/api/ml/predict-delay',
        json={
            'project_id': 'proj001',
            'tasks_total': 50,
            'tasks_completed': 20,
            'days_elapsed': 30,
            'days_remaining': 20,
            'team_size': 5,
            'blocked_tasks': 3
        }
    )
    result = response.json()
    # Returns: {'delay_probability': 0.72, 'estimated_delay_days': 5, 'factors': [...]}
    return result
```

## Frontend Integration Patterns

### React Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';
import axios from 'axios';

const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('authToken'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      // Verify token and get user data
      axios.get(`${process.env.REACT_APP_API_URL}/auth/me`, {
        headers: { Authorization: `Bearer ${token}` }
      })
      .then(res => setUser(res.data))
      .catch(() => logout())
      .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    setToken(token);
    setUser(user);
    localStorage.setItem('authToken', token);
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('authToken');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### Task Board Component

```javascript
// components/TaskBoard.jsx
import { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../hooks/useAuth';

export function TaskBoard() {
  const { token } = useAuth();
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    
    const grouped = response.data.reduce((acc, task) => {
      acc[task.status].push(task);
      return acc;
    }, { todo: [], in_progress: [], done: [] });
    
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, 
      { status: newStatus },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    loadTasks();
  };

  return (
    <div className="task-board">
      {['todo', 'in_progress', 'done'].map(status => (
        <div key={status} className="task-column">
          <h3>{status.replace('_', ' ').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <TaskCard 
              key={task._id} 
              task={task} 
              onMove={moveTask} 
            />
          ))}
        </div>
      ))}
    </div>
  );
}
```

### AI Analytics Dashboard

```javascript
// components/AdminAnalytics.jsx
import { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../hooks/useAuth';

export function AdminAnalytics() {
  const { token } = useAuth();
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    loadAnalytics();
    checkUserRisks();
  }, []);

  const loadAnalytics = async () => {
    const response = await axios.get(`${process.env.REACT_APP_API_URL}/analytics/overview`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    setAnalytics(response.data);
  };

  const checkUserRisks = async () => {
    const usersResponse = await axios.get(`${process.env.REACT_APP_API_URL}/users`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    
    const riskPromises = usersResponse.data.map(async (user) => {
      const riskResponse = await axios.post(`${process.env.REACT_APP_ML_API_URL}/api/ml/predict-risk`, {
        user_id: user._id,
        login_frequency: user.stats.loginFrequency,
        failed_logins: user.stats.failedLogins,
        unusual_hours: user.stats.unusualHours,
        data_access_volume: user.stats.dataAccess
      });
      
      return {
        user: user.username,
        risk: riskResponse.data
      };
    });
    
    const risks = await Promise.all(riskPromises);
    setRiskUsers(risks.filter(r => r.risk.risk_level !== 'low'));
  };

  return (
    <div className="analytics-dashboard">
      <h2>Organization Analytics</h2>
      {analytics && (
        <div className="metrics">
          <div>Total Users: {analytics.totalUsers}</div>
          <div>Active Tasks: {analytics.activeTasks}</div>
          <div>Open Tickets: {analytics.openTickets}</div>
        </div>
      )}
      
      <h3>High Risk Users</h3>
      <table>
        <thead>
          <tr>
            <th>User</th>
            <th>Risk Level</th>
            <th>Score</th>
          </tr>
        </thead>
        <tbody>
          {riskUsers.map((item, idx) => (
            <tr key={idx}>
              <td>{item.user}</td>
              <td className={`risk-${item.risk.risk_level}`}>
                {item.risk.risk_level}
              </td>
              <td>{(item.risk.risk_score * 100).toFixed(1)}%</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/dbname

# JWT Authentication
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRE=24h
REFRESH_TOKEN_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://ml-service:8000

# Email (for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Rate Limiting
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX=100
```

### ML Service Configuration

```env
# API Settings
API_HOST=0.0.0.0
API_PORT=8000
LOG_LEVEL=INFO

# Model Paths
MODEL_PATH=./models
TICKET_CLASSIFIER_PATH=./models/ticket_classifier.pkl
RISK_MODEL_PATH=./models/risk_predictor.pkl

# ML Parameters
ANOMALY_THRESHOLD=0.75
BURNOUT_THRESHOLD=0.70
CONFIDENCE_THRESHOLD=0.60
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

export function ProtectedRoute({ children, adminOnly = false }) {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
}

// App.jsx usage
<Route path="/admin" element={
  <ProtectedRoute adminOnly={true}>
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### API Client with Interceptors

```javascript
// utils/apiClient.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add token to all requests
apiClient.interceptors.request.use(
  config => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

// Handle 401 errors
apiClient.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Real-time Notifications

```javascript
// hooks/useNotifications.js
import { useState, useEffect } from 'react';
import io from 'socket.io-client';
import { useAuth } from './useAuth';

export function useNotifications() {
  const { token, user } = useAuth();
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    if (!token || !user) return;

    const socket = io(process.env.REACT_APP_API_URL, {
      auth: { token }
    });

    socket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    return () => socket.disconnect();
  }, [token, user]);

  const markAsRead = (notificationId) => {
    setNotifications(prev => 
      prev.map(n => n.id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
}
```

## Troubleshooting

### JWT Token Expired

**Problem:** Getting 401 errors even when logged in.

**Solution:** Implement token refresh:

```javascript
async function refreshToken() {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
    refreshToken
  });
  localStorage.setItem('authToken', response.data.token);
  return response.data.token;
}
```

### ML Service Connection Issues

**Problem:** Frontend cannot reach ML service.

**Solution:** Ensure ML service is proxied through backend:

```javascript
// backend/routes/ml.js
const axios = require('axios');

router.post('/classify-ticket', async (req, res) => {
  try {
    const mlResponse = await axios.post(
      `${process.env.ML_SERVICE_URL}/api/ml/classify-ticket`,
      req.body,
      { timeout: 10000 }
    );
    res.json(mlResponse.data);
  } catch (error) {
    console.error('ML service error:', error.message);
    res.status(503).json({ error: 'ML service unavailable' });
  }
});
```

### MongoDB Connection Failures

**Problem:** Backend cannot connect to MongoDB.

**Solution:** Check connection string and add retry logic:

```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  const options = {
    useNewUrlParser: true,
    useUnifiedTopology: true,
    serverSelectionTimeoutMS: 5000,
    retryWrites: true
  };

  try {
    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};
```

### Model Training for ML Service

**Problem:** Need to train custom models.

**Solution:** Use provided training scripts:

```python
# ml-service/train_models.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import pickle

def train_ticket_classifier(training_data):
    X = training_data['features']
    y = training_data['labels']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    print(f'Model accuracy: {accuracy:.2f}')
    
    # Save model
    with open('models/ticket_classifier.pkl', 'wb') as f:
        pickle.dump(model, f)

if __name__ == '__main__':
    # Load your training data
    data = load_training_data()
    train_ticket_classifier(data)
```

### CORS Issues

**Problem:** Frontend blocked by CORS policy.

**Solution:** Configure CORS in backend:

```javascript
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```
