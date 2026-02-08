import express from 'express';
import cors from 'cors';

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

// In-memory storage
const storage = {
    wallets: new Map(),
    transactions: []
};

function generateTxId() {
    return `TXN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}

function getWallet(userId) {
    if (!storage.wallets.has(userId)) {
        storage.wallets.set(userId, {
            userId,
            balance: 0,
            frozenBalance: 0,
            totalRecharge: 0,
            totalSpent: 0,
            totalEarning: 0,
            totalWithdrawal: 0
        });
    }
    return storage.wallets.get(userId);
}

// Health check
app.get('/health', (req, res) => {
    res.json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime()
    });
});

// Admin recharge
app.post('/api/wallet/admin/recharge', (req, res) => {
    try {
        const { userId, amount, operatorId, remark } = req.body;
        
        if (!userId || !amount) {
            return res.status(400).json({
                success: false,
                error: 'Missing required fields'
            });
        }
        
        const wallet = getWallet(userId);
        const balanceBefore = wallet.balance;
        wallet.balance += amount;
        wallet.totalRecharge += amount;
        
        const transaction = {
            transactionId: generateTxId(),
            userId,
            type: 'RECHARGE',
            amount,
            balanceBefore,
            balanceAfter: wallet.balance,
            operatorId,
            remark: remark || `Admin recharge ${amount} points`,
            createdAt: new Date().toISOString()
        };
        
        storage.transactions.push(transaction);
        
        res.json({
            success: true,
            message: `Successfully recharged ${amount} points`,
            data: {
                userId,
                balance: wallet.balance,
                transaction
            }
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

// Get wallet
app.get('/api/wallet/:userId', (req, res) => {
    try {
        const { userId } = req.params;
        const wallet = getWallet(userId);
        
        res.json({
            success: true,
            data: wallet
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

// Get transactions
app.get('/api/wallet/:userId/transactions', (req, res) => {
    try {
        const { userId } = req.params;
        const userTransactions = storage.transactions.filter(tx => tx.userId === userId);
        
        res.json({
            success: true,
            transactions: userTransactions,
            total: userTransactions.length
        });
    } catch (error) {
        res.status(500).json({
            success: false,
            error: error.message
        });
    }
});

app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});# mlbb-backend
