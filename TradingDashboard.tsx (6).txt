'use client'

import React, { useState, useEffect, useRef } from 'react'
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"
import { Label } from "@/components/ui/label"
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs"
import { Switch } from "@/components/ui/switch"
import { Home, Plug, Settings, Upload } from 'lucide-react'
import MetaApi from 'metaapi.cloud-sdk'

export default function TradingDashboard() {
  const [activeTab, setActiveTab] = useState('authentication')
  const [isTrading, setIsTrading] = useState(false)
  const [logs, setLogs] = useState<string[]>([])
  const [botImage, setBotImage] = useState('/placeholder.svg')
  const [isConnected, setIsConnected] = useState(false)
  const [tradingInterval, setTradingInterval] = useState<NodeJS.Timeout | null>(null)
  const totalTrades = 6
  const [tradesOpened, setTradesOpened] = useState(0)
  const metaApiRef = useRef<MetaApi | null>(null)
  const connectionRef = useRef<any>(null)

  const [authForm, setAuthForm] = useState({ mentorId: '', email: '', licenseKey: '' })
  const [connectForm, setConnectForm] = useState({ apiToken: '', accountId: '' })
  const [settings, setSettings] = useState({
    riskPerTrade: 2,
    stopLoss: 10,
    takeProfitMultiplier: 2,
    tradeSize: 0.01,
    tradingPairs: 'XAUUSD', // Initial value, modifiable by user
    copyAllTrades: true,
    enableNotifications: true
  })

  useEffect(() => {
    return () => {
      if (tradingInterval) clearInterval(tradingInterval)
      if (connectionRef.current) connectionRef.current.close()
    }
  }, [])

  const addLog = (message: string) => {
    setLogs(prevLogs => [...prevLogs, message])
  }

  const authenticateUser = (e: React.FormEvent) => {
    e.preventDefault()
    if (validateCredentials(authForm.mentorId, authForm.email, authForm.licenseKey)) {
      setActiveTab('home')
      addLog('Authentication successful for Mentor ID: ' + authForm.mentorId)
    } else {
      alert('Invalid Mentor ID, Email, or License Key.')
    }
  }

  const validateCredentials = (mentorId: string, email: string, licenseKey: string) => {
    return mentorId && email && licenseKey
  }

  const toggleTrade = () => {
    setIsTrading(prev => !prev)
    if (!isTrading) {
      addLog("Trading started...")
      startTradingLogs()
    } else {
      if (tradingInterval) clearInterval(tradingInterval)
      addLog("Trading stopped.")
      setTradesOpened(0)
    }
  }

  const startTradingLogs = () => {
    addLog("Verifying account....");
    setTimeout(() => addLog("Account details successfully submitted....."), 2000);
    setTimeout(() => addLog("Fetching trading symbols..."), 4000);

    const pairs = settings.tradingPairs.split(',').map(pair => pair.trim().toUpperCase())
    let tradeCount = 0;
    const interval = setInterval(() => {
      if (tradeCount < totalTrades && pairs.length > 0) {
        const pair = pairs[tradeCount % pairs.length] // Cycle through pairs
        placeTrade(pair);
        tradeCount++;
        setTradesOpened(tradeCount);
      } else {
        clearInterval(interval);
        addLog("All trades have been placed.");
        setIsTrading(false);
      }
    }, 6000);

    setTradingInterval(interval);
  }

  const connectToAccount = async (e: React.FormEvent) => {
    e.preventDefault()
    addLog('Connecting to MetaApi...')
    try {
      metaApiRef.current = new MetaApi(connectForm.apiToken)
      const account = await metaApiRef.current.metatraderAccountApi.getAccount(connectForm.accountId)
      if (!account.connectionStatus || account.connectionStatus === 'DISCONNECTED') {
        await account.deploy()
      }
      await account.waitConnected()
      connectionRef.current = account.getRPCConnection()
      await connectionRef.current.connect()
      addLog('Connected to MetaApi successfully')
      setIsConnected(true)
      setActiveTab('home')
    } catch (error) {
      addLog(`Error connecting to MetaApi: ${error.message}`)
      console.error('Full error:', error)
    }
  }

  const placeTrade = async (symbol: string) => {
    if (!isConnected || !connectionRef.current) {
      addLog(`Not connected to MetaApi. Cannot place trade for ${symbol}.`);
      return;
    }

    try {
      const price = await connectionRef.current.getSymbolPrice(symbol);
      const stopLossPrice = price.bid - 50;
      const takeProfitPrice = price.ask + 100;

      const result = await connectionRef.current.createMarketBuyOrder(
        symbol,
        settings.tradeSize,
        stopLossPrice,
        takeProfitPrice,
        { comment: 'C.O.N-IBOT-MOBILE' }
      );
      addLog(`Trade placed successfully for ${symbol}. Order ID: ${result.orderId}`);
      addLog(`BUY ${symbol}, Volume: ${settings.tradeSize}, SL: ${stopLossPrice.toFixed(2)}, TP: ${takeProfitPrice.toFixed(2)}`);
    } catch (error) {
      addLog(`Error placing trade for ${symbol}: ${error.message}`);
    }
  }

  const saveSettings = () => {
    addLog(`Settings saved: Risk ${settings.riskPerTrade}%, Stop Loss ${settings.stopLoss}%, TP Multiplier ${settings.takeProfitMultiplier}, Trade Size ${settings.tradeSize}, Pairs: ${settings.tradingPairs}, Copy Trades: ${settings.copyAllTrades}, Notifications: ${settings.enableNotifications}`)
    alert('Settings saved successfully!')
  }

  const handleImageUpload = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0]
    if (file) {
      const reader = new FileReader()
      reader.onload = (e) => {
        setBotImage(e.target?.result as string)
      }
      reader.readAsDataURL(file)
    }
  }

  return (
    <div className="flex flex-col min-h-screen bg-background text-foreground">
      <main className="flex-grow p-6">
        <Tabs value={activeTab} onValueChange={setActiveTab}>
          <TabsList className="grid w-full grid-cols-3">
            <TabsTrigger value="authentication">Authentication</TabsTrigger>
            <TabsTrigger value="home">Home</TabsTrigger>
            <TabsTrigger value="connect">Connect</TabsTrigger>
          </TabsList>

          <TabsContent value="authentication">
            <Card>
              <CardHeader>
                <CardTitle>Authentication</CardTitle>
              </CardHeader>
              <CardContent>
                <form onSubmit={authenticateUser} className="space-y-4">
                  <div className="space-y-2">
                    <Label htmlFor="mentorId">Mentor ID</Label>
                    <Input
                      id="mentorId"
                      value={authForm.mentorId}
                      onChange={(e) => setAuthForm({ ...authForm, mentorId: e.target.value })}
                      required
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="email">Email</Label>
                    <Input
                      id="email"
                      type="email"
                      value={authForm.email}
                      onChange={(e) => setAuthForm({ ...authForm, email: e.target.value })}
                      required
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="licenseKey">License Key</Label>
                    <Input
                      id="licenseKey"
                      value={authForm.licenseKey}
                      onChange={(e) => setAuthForm({ ...authForm, licenseKey: e.target.value })}
                      required
                    />
                  </div>
                  <Button type="submit">Authenticate</Button>
                </form>
              </CardContent>
            </Card>
          </TabsContent>

          <TabsContent value="home">
            <Card>
              <CardHeader>
                <CardTitle>CHILD OF NASDAQ</CardTitle>
              </CardHeader>
              <CardContent className="space-y-4">
                <div className="flex flex-col items-center">
                  <img src={botImage} alt="Bot" className="w-64 h-64 object-cover rounded-lg mb-4" />
                  <p>C.O.N-IBOT, 24/7 OPERATION</p>
                  <Button onClick={() => document.getElementById('fileInput')?.click()}>
                    <Upload className="mr-2 h-4 w-4" /> Upload Icon
                  </Button>
                  <input
                    type="file"
                    id="fileInput"
                    className="hidden"
                    accept="image/*"
                    onChange={handleImageUpload}
                  />
                </div>
                <div className="bg-secondary p-4 rounded-lg relative">
                  <div className="absolute top-2 left-2 w-2 h-2 bg-green-500 rounded-full animate-pulse" />
                  <h3 className="font-bold mb-2">Bot Logs</h3>
                  <ul className="space-y-1 max-h-40 overflow-y-auto">
                    {logs.map((log, index) => (
                      <li key={index} className="text-sm text-green-500">{log}</li>
                    ))}
                  </ul>
                </div>
                <div className="flex justify-between">
                  <Button onClick={toggleTrade} variant={isTrading ? "destructive" : "default"}>
                    {isTrading ? "Stop Trading" : "Start Trading"}
                  </Button>
                  <Button variant="outline" onClick={() => setActiveTab('settings')}>Settings</Button>
                </div>
              </CardContent>
            </Card>
          </TabsContent>

          <TabsContent value="connect">
            <Card>
              <CardHeader>
                <CardTitle>Connect MetaApi Account</CardTitle>
              </CardHeader>
              <CardContent>
                <form onSubmit={connectToAccount} className="space-y-4">
                  <div className="space-y-2">
                    <Label htmlFor="apiToken">MetaApi Token</Label>
                    <Input
                      id="apiToken"
                      value={connectForm.apiToken}
                      onChange={(e) => setConnectForm({ ...connectForm, apiToken: e.target.value })}
                      required
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="accountId">MetaApi Account ID</Label>
                    <Input
                      id="accountId"
                      value={connectForm.accountId}
                      onChange={(e) => setConnectForm({ ...connectForm, accountId: e.target.value })}
                      required
                    />
                  </div>
                  <Button type="submit">Connect</Button>
                </form>
              </CardContent>
            </Card>
          </TabsContent>

          <TabsContent value="settings">
            <Card>
              <CardHeader>
                <CardTitle>Risk Management Settings</CardTitle>
              </CardHeader>
              <CardContent>
                <form onSubmit={(e) => { e.preventDefault(); saveSettings(); }} className="space-y-4">
                  <div className="space-y-2">
                    <Label htmlFor="riskPerTrade">Max Risk Per Trade (%)</Label>
                    <Input
                      id="riskPerTrade"
                      type="number"
                      value={settings.riskPerTrade}
                      onChange={(e) => setSettings({ ...settings, riskPerTrade: parseFloat(e.target.value) })}
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="stopLoss">Stop Loss (pips)</Label>
                    <Input
                      id="stopLoss"
                      type="number"
                      value={settings.stopLoss}
                      onChange={(e) => setSettings({ ...settings, stopLoss: parseFloat(e.target.value) })}
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="takeProfitMultiplier">Take Profit Multiplier</Label>
                    <Input
                      id="takeProfitMultiplier"
                      type="number"
                      value={settings.takeProfitMultiplier}
                      onChange={(e) => setSettings({ ...settings, takeProfitMultiplier: parseFloat(e.target.value) })}
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="tradeSize">Trade Size (Lots)</Label>
                    <Input
                      id="tradeSize"
                      type="number"
                      value={settings.tradeSize}
                      onChange={(e) => setSettings({ ...settings, tradeSize: parseFloat(e.target.value) })}
                    />
                  </div>
                  <div className="space-y-2">
                    <Label htmlFor="tradingPairs">Trading Pairs (comma-separated)</Label>
                    <Input
                      id="tradingPairs"
                      value={settings.tradingPairs}
                      onChange={(e) => setSettings({ ...settings, tradingPairs: e.target.value })}
                    />
                  </div>
                  <div className="flex items-center space-x-2">
                    <Switch
                      id="copyAllTrades"
                      checked={settings.copyAllTrades}
                      onCheckedChange={(checked) => setSettings({ ...settings, copyAllTrades: checked })}
                    />
                    <Label htmlFor="copyAllTrades">Copy All Trades</Label>
                  </div>
                  <div className="flex items-center space-x-2">
                    <Switch
                      id="enableNotifications"
                      checked={settings.enableNotifications}
                      onCheckedChange={(checked) => setSettings({ ...settings, enableNotifications: checked })}
                    />
                    <Label htmlFor="enableNotifications">Enable Notifications</Label>
                  </div>
                  <Button type="submit">Save Settings</Button>
                </form>
              </CardContent>
            </Card>
          </TabsContent>
        </Tabs>
      </main>

      <footer className="bg-secondary p-4">
        <div className="flex justify-around">
          <Button variant="ghost" onClick={() => setActiveTab('home')}>
            <Home className="h-5 w-5" />
            <span className="sr-only">Home</span>
          </Button>
          <Button variant="ghost" onClick={() => setActiveTab('connect')}>
            <Plug className="h-5 w-5" />
            <span className="sr-only">Connect</span>
          </Button>
          <Button variant="ghost" onClick={() => setActiveTab('settings')}>
            <Settings className="h-5 w-5" />
            <span className="sr-only">Settings</span>
          </Button>
        </div>
      </footer>
    </div>
  )
}

