# OPTYMA AGRITECH PROTOCOL - Frontend Architecture

## 🎨 Arsitektur Frontend

### Tech Stack Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend Architecture                    │
├─────────────────────────────────────────────────────────────┤
│  Next.js 15 (App Router) + TypeScript + Tailwind CSS      │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Web3      │  │    State    │  │     UI      │         │
│  │ Integration │  │ Management  │  │ Components  │         │
│  │             │  │             │  │             │         │
│  │ • Ethers.js │  │ • Zustand   │  │ • Ant Design│         │
│  │ • Wagmi     │  │ • React Query│  │ • Tailwind │         │
│  │ • RainbowKit│  │ • SWR       │  │ • Custom    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## 📁 Project Structure

```
optyma-frontend/
├── app/                           # Next.js 15 App Router
│   ├── (auth)/                   # Authentication routes
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/              # Dashboard routes
│   │   ├── agridaos/
│   │   ├── consultants/
│   │   ├── knowledge/
│   │   └── profile/
│   ├── api/                      # API routes
│   │   ├── auth/
│   │   ├── blockchain/
│   │   └── ai-agents/
│   ├── globals.css
│   ├── layout.tsx
│   └── page.tsx
├── components/                    # Reusable components
│   ├── ui/                       # Base UI components
│   ├── web3/                     # Web3 specific components
│   ├── forms/                    # Form components
│   └── layout/                   # Layout components
├── hooks/                        # Custom React hooks
│   ├── useContract.ts
│   ├── useAgriDAO.ts
│   └── useAuth.ts
├── lib/                          # Utility libraries
│   ├── contracts/                # Contract ABIs and addresses
│   ├── utils/
│   └── validations/
├── store/                        # State management
│   ├── authStore.ts
│   ├── web3Store.ts
│   └── agriDAOStore.ts
├── types/                        # TypeScript type definitions
└── public/                       # Static assets
```

## 🔧 Core Components

### 1. Layout Structure

```typescript
// app/layout.tsx
'use client';

import { Inter } from 'next/font/google';
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { RainbowKitProvider } from '@rainbow-me/rainbowkit';
import { config } from '@/lib/wagmi';
import { Toaster } from 'react-hot-toast';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });
const queryClient = new QueryClient();

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="id">
      <body className={inter.className}>
        <WagmiProvider config={config}>
          <QueryClientProvider client={queryClient}>
            <RainbowKitProvider>
              <div className="min-h-screen bg-gradient-to-br from-green-50 to-blue-50">
                <Navbar />
                <main>{children}</main>
                <Footer />
                <Toaster position="top-right" />
              </div>
            </RainbowKitProvider>
          </QueryClientProvider>
        </WagmiProvider>
      </body>
    </html>
  );
}
```

### 2. Web3 Configuration

```typescript
// lib/wagmi.ts
import { getDefaultConfig } from '@rainbow-me/rainbowkit';
import { polygon, polygonMumbai } from 'wagmi/chains';

export const config = getDefaultConfig({
  appName: 'Optyma Agritech Protocol',
  projectId: process.env.NEXT_PUBLIC_WALLET_CONNECT_PROJECT_ID!,
  chains: [polygon, polygonMumbai],
  ssr: true,
});

// Contract addresses
export const CONTRACT_ADDRESSES = {
  [polygon.id]: {
    OPTYMA_TOKEN: '0x...',
    AGRIDAO_FACTORY: '0x...',
    KNOWLEDGE_TOKENS: '0x...',
    ANTI_SCAM_REGISTRY: '0x...',
  },
  [polygonMumbai.id]: {
    OPTYMA_TOKEN: '0x...',
    AGRIDAO_FACTORY: '0x...',
    KNOWLEDGE_TOKENS: '0x...',
    ANTI_SCAM_REGISTRY: '0x...',
  },
};
```

### 3. Custom Hooks

```typescript
// hooks/useContract.ts
import { useReadContract, useWriteContract } from 'wagmi';
import { useChainId } from 'wagmi';
import { CONTRACT_ADDRESSES } from '@/lib/wagmi';
import { AGRIDAO_FACTORY_ABI } from '@/lib/contracts/abis';

export function useAgriDAOFactory() {
  const chainId = useChainId();
  const contractAddress = CONTRACT_ADDRESSES[chainId]?.AGRIDAO_FACTORY;

  const { data: daoCounter } = useReadContract({
    address: contractAddress,
    abi: AGRIDAO_FACTORY_ABI,
    functionName: 'daoCounter',
  });

  const { writeContract: createDAO, isPending: isCreating } = useWriteContract();

  const createAgriDAO = async (params: {
    name: string;
    symbol: string;
    specialty: string;
    consultants: string[];
    targetRaise: bigint;
  }) => {
    if (!contractAddress) throw new Error('Contract not deployed on this chain');
    
    return writeContract({
      address: contractAddress,
      abi: AGRIDAO_FACTORY_ABI,
      functionName: 'createAgriDAO',
      args: [
        params.name,
        params.symbol,
        params.specialty,
        params.consultants,
        params.targetRaise,
      ],
    });
  };

  return {
    daoCounter,
    createAgriDAO,
    isCreating,
  };
}

// hooks/useAgriDAO.ts
export function useAgriDAO(daoAddress: string) {
  const { data: daoInfo } = useReadContract({
    address: daoAddress as `0x${string}`,
    abi: AGRIDAO_ABI,
    functionName: 'specialty',
  });

  const { writeContract: proposeProject } = useWriteContract();

  const createProject = async (params: {
    title: string;
    description: string;
    fundingGoal: bigint;
    deadline: bigint;
  }) => {
    return proposeProject({
      address: daoAddress as `0x${string}`,
      abi: AGRIDAO_ABI,
      functionName: 'proposeProject',
      args: [params.title, params.description, params.fundingGoal, params.deadline],
    });
  };

  return {
    daoInfo,
    createProject,
  };
}
```

### 4. State Management

```typescript
// store/web3Store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface Web3State {
  isConnected: boolean;
  address: string | null;
  chainId: number | null;
  balance: string;
  setConnection: (connected: boolean, address?: string, chainId?: number) => void;
  setBalance: (balance: string) => void;
  disconnect: () => void;
}

export const useWeb3Store = create<Web3State>()(
  persist(
    (set) => ({
      isConnected: false,
      address: null,
      chainId: null,
      balance: '0',
      setConnection: (connected, address, chainId) =>
        set({ isConnected: connected, address, chainId }),
      setBalance: (balance) => set({ balance }),
      disconnect: () =>
        set({ isConnected: false, address: null, chainId: null, balance: '0' }),
    }),
    {
      name: 'web3-storage',
    }
  )
);

// store/agriDAOStore.ts
interface AgriDAO {
  id: number;
  address: string;
  name: string;
  specialty: string;
  consultants: string[];
  currentRaise: string;
  targetRaise: string;
  isActive: boolean;
}

interface AgriDAOState {
  daos: AgriDAO[];
  selectedDAO: AgriDAO | null;
  loading: boolean;
  setDAOs: (daos: AgriDAO[]) => void;
  selectDAO: (dao: AgriDAO) => void;
  setLoading: (loading: boolean) => void;
}

export const useAgriDAOStore = create<AgriDAOState>((set) => ({
  daos: [],
  selectedDAO: null,
  loading: false,
  setDAOs: (daos) => set({ daos }),
  selectDAO: (dao) => set({ selectedDAO: dao }),
  setLoading: (loading) => set({ loading }),
}));
```

## 📱 Key Pages & Components

### 1. Dashboard Utama

```typescript
// app/(dashboard)/page.tsx
'use client';

import { useAccount } from 'wagmi';
import { useAgriDAOStore } from '@/store/agriDAOStore';
import { StatsCard } from '@/components/dashboard/StatsCard';
import { DAOCard } from '@/components/dashboard/DAOCard';
import { RecentActivity } from '@/components/dashboard/RecentActivity';

export default function Dashboard() {
  const { address } = useAccount();
  const { daos, loading } = useAgriDAOStore();

  if (!address) {
    return <ConnectWalletPrompt />;
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900 mb-2">
          Dashboard Optyma Protocol
        </h1>
        <p className="text-gray-600">
          Kelola investasi dan partisipasi Anda dalam ekosistem pertanian terdesentralisasi
        </p>
      </div>

      {/* Statistics */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-6 mb-8">
        <StatsCard
          title="Total AgriDAOs"
          value={daos.length}
          icon="🚜"
          trend="+12%"
        />
        <StatsCard
          title="Portfolio Value"
          value="$12,450"
          icon="💰"
          trend="+5.4%"
        />
        <StatsCard
          title="Consultations"
          value="23"
          icon="👨‍🌾"
          trend="+8"
        />
        <StatsCard
          title="Knowledge Tokens"
          value="7"
          icon="📚"
          trend="+2"
        />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
        {/* AgriDAOs */}
        <div className="lg:col-span-2">
          <div className="bg-white rounded-lg shadow-sm p-6">
            <h2 className="text-xl font-semibold mb-4">AgriDAOs Terbaru</h2>
            <div className="space-y-4">
              {loading ? (
                <SkeletonLoader />
              ) : (
                daos.map((dao) => (
                  <DAOCard key={dao.id} dao={dao} />
                ))
              )}
            </div>
          </div>
        </div>

        {/* Recent Activity */}
        <div>
          <RecentActivity />
        </div>
      </div>
    </div>
  );
}
```

### 2. AgriDAO Creation Form

```typescript
// components/forms/CreateDAOForm.tsx
'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useAgriDAOFactory } from '@/hooks/useContract';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';
import { Select } from '@/components/ui/Select';
import { Textarea } from '@/components/ui/Textarea';

const createDAOSchema = z.object({
  name: z.string().min(3, 'Nama minimal 3 karakter'),
  symbol: z.string().min(2, 'Symbol minimal 2 karakter').max(6, 'Symbol maksimal 6 karakter'),
  specialty: z.enum(['aquaculture', 'livestock', 'crops', 'probiotics', 'supply']),
  description: z.string().min(10, 'Deskripsi minimal 10 karakter'),
  targetRaise: z.string().min(1, 'Target funding harus diisi'),
  consultants: z.array(z.string()).min(1, 'Minimal 1 konsultan'),
});

type CreateDAOForm = z.infer<typeof createDAOSchema>;

export function CreateDAOForm() {
  const [step, setStep] = useState(1);
  const { createAgriDAO, isCreating } = useAgriDAOFactory();

  const {
    register,
    handleSubmit,
    formState: { errors },
    watch,
    setValue,
  } = useForm<CreateDAOForm>({
    resolver: zodResolver(createDAOSchema),
  });

  const onSubmit = async (data: CreateDAOForm) => {
    try {
      const targetRaise = parseEther(data.targetRaise);
      
      await createAgriDAO({
        name: data.name,
        symbol: data.symbol,
        specialty: data.specialty,
        consultants: data.consultants,
        targetRaise,
      });

      toast.success('AgriDAO berhasil dibuat!');
    } catch (error) {
      toast.error('Gagal membuat AgriDAO');
      console.error(error);
    }
  };

  return (
    <div className="max-w-2xl mx-auto bg-white rounded-lg shadow-lg p-8">
      <div className="mb-8">
        <h2 className="text-2xl font-bold text-gray-900 mb-2">
          Buat AgriDAO Baru
        </h2>
        <p className="text-gray-600">
          Langkah {step} dari 3: Informasi Dasar
        </p>
      </div>

      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        {step === 1 && (
          <>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Nama AgriDAO
              </label>
              <Input
                {...register('name')}
                placeholder="contoh: Udang Vaname Indonesia"
                error={errors.name?.message}
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Symbol Token
              </label>
              <Input
                {...register('symbol')}
                placeholder="contoh: UVI"
                className="uppercase"
                error={errors.symbol?.message}
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Spesialisasi
              </label>
              <Select
                {...register('specialty')}
                options={[
                  { value: 'aquaculture', label: 'Budidaya Perairan (Aquaculture)' },
                  { value: 'livestock', label: 'Peternakan (Livestock)' },
                  { value: 'crops', label: 'Pertanian Tanaman (Crops)' },
                  { value: 'probiotics', label: 'Probiotik & R&D' },
                  { value: 'supply', label: 'Supply Chain' },
                ]}
                error={errors.specialty?.message}
              />
            </div>

            <Button
              type="button"
              onClick={() => setStep(2)}
              className="w-full"
            >
              Lanjut ke Detail
            </Button>
          </>
        )}

        {step === 2 && (
          <>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Deskripsi AgriDAO
              </label>
              <Textarea
                {...register('description')}
                rows={4}
                placeholder="Jelaskan tujuan dan visi AgriDAO Anda..."
                error={errors.description?.message}
              />
            </div>

            <div>
              <label className="block text-sm font-medium text-gray-700 mb-2">
                Target Funding (ETH)
              </label>
              <Input
                {...register('targetRaise')}
                type="number"
                step="0.01"
                placeholder="10.0"
                error={errors.targetRaise?.message}
              />
            </div>

            <div className="flex gap-4">
              <Button
                type="button"
                variant="outline"
                onClick={() => setStep(1)}
                className="flex-1"
              >
                Kembali
              </Button>
              <Button
                type="button"
                onClick={() => setStep(3)}
                className="flex-1"
              >
                Lanjut ke Konsultan
              </Button>
            </div>
          </>
        )}

        {step === 3 && (
          <>
            <ConsultantSelector
              onConsultantsChange={(consultants) => 
                setValue('consultants', consultants)
              }
            />

            <div className="flex gap-4">
              <Button
                type="button"
                variant="outline"
                onClick={() => setStep(2)}
                className="flex-1"
              >
                Kembali
              </Button>
              <Button
                type="submit"
                disabled={isCreating}
                className="flex-1"
              >
                {isCreating ? 'Membuat...' : 'Buat AgriDAO'}
              </Button>
            </div>
          </>
        )}
      </form>
    </div>
  );
}
```

### 3. Consultation Booking Component

```typescript
// components/consultation/BookingForm.tsx
'use client';

import { useState } from 'react';
import { useAgriDAO } from '@/hooks/useContract';
import { parseEther } from 'viem';

interface Consultant {
  address: string;
  name: string;
  expertise: string;
  yearsExperience: number;
  consultationFee: string;
  rating: number;
  isVerified: boolean;
}

interface BookingFormProps {
  consultant: Consultant;
  daoAddress: string;
}

export function BookingForm({ consultant, daoAddress }: BookingFormProps) {
  const [selectedTime, setSelectedTime] = useState<string>('');
  const [topic, setTopic] = useState<string>('');
  const { bookConsultation, isBooking } = useAgriDAO(daoAddress);

  const handleBooking = async () => {
    try {
      const fee = parseEther(consultant.consultationFee);
      
      await bookConsultation(consultant.address, { value: fee });
      
      // Here you would also create a consultation session
      // and send calendar invite
      
      toast.success('Konsultasi berhasil dibooking!');
    } catch (error) {
      toast.error('Gagal booking konsultasi');
    }
  };

  return (
    <div className="bg-white rounded-lg border p-6">
      <div className="flex items-center gap-4 mb-6">
        <img
          src={`https://api.dicebear.com/7.x/avataaars/svg?seed=${consultant.address}`}
          alt={consultant.name}
          className="w-16 h-16 rounded-full"
        />
        <div>
          <h3 className="text-lg font-semibold">{consultant.name}</h3>
          <p className="text-gray-600">{consultant.expertise}</p>
          <div className="flex items-center gap-2 mt-1">
            <span className="text-yellow-500">⭐</span>
            <span className="text-sm">{consultant.rating}/5</span>
            {consultant.isVerified && (
              <span className="bg-green-100 text-green-800 text-xs px-2 py-1 rounded">
                Verified
              </span>
            )}
          </div>
        </div>
      </div>

      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Topik Konsultasi
          </label>
          <input
            type="text"
            value={topic}
            onChange={(e) => setTopic(e.target.value)}
            placeholder="contoh: Penyakit udang vaname"
            className="w-full px-3 py-2 border border-gray-300 rounded-md"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Waktu Preferred
          </label>
          <select
            value={selectedTime}
            onChange={(e) => setSelectedTime(e.target.value)}
            className="w-full px-3 py-2 border border-gray-300 rounded-md"
          >
            <option value="">Pilih waktu</option>
            <option value="morning">Pagi (08:00-12:00)</option>
            <option value="afternoon">Siang (13:00-17:00)</option>
            <option value="evening">Malam (19:00-21:00)</option>
          </select>
        </div>

        <div className="border-t pt-4">
          <div className="flex justify-between items-center mb-4">
            <span className="text-gray-600">Biaya Konsultasi:</span>
            <span className="text-lg font-semibold">
              {consultant.consultationFee} ETH
            </span>
          </div>
          
          <Button
            onClick={handleBooking}
            disabled={!selectedTime || !topic || isBooking}
            className="w-full"
          >
            {isBooking ? 'Memproses...' : 'Book Konsultasi'}
          </Button>
        </div>
      </div>
    </div>
  );
}
```

## 🎨 UI Component Library

### Base Components

```typescript
// components/ui/Button.tsx
import { cn } from '@/lib/utils';
import { ButtonHTMLAttributes, forwardRef } from 'react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = 'primary', size = 'md', ...props }, ref) => {
    return (
      <button
        ref={ref}
        className={cn(
          'inline-flex items-center justify-center rounded-md font-medium transition-colors',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
          'disabled:pointer-events-none disabled:opacity-50',
          {
            'bg-green-600 text-white hover:bg-green-700': variant === 'primary',
            'bg-gray-100 text-gray-900 hover:bg-gray-200': variant === 'secondary',
            'border border-gray-300 bg-transparent hover:bg-gray-50': variant === 'outline',
            'hover:bg-gray-100': variant === 'ghost',
          },
          {
            'h-8 px-3 text-sm': size === 'sm',
            'h-10 px-4': size === 'md',
            'h-12 px-6 text-lg': size === 'lg',
          },
          className
        )}
        {...props}
      />
    );
  }
);
```

## 📱 Responsive Design

### Mobile-First Approach

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  * {
    @apply border-border;
  }
  
  body {
    @apply bg-background text-foreground;
  }
}

@layer components {
  .container {
    @apply mx-auto px-4 sm:px-6 lg:px-8;
  }
  
  .card {
    @apply bg-white rounded-lg shadow-sm border border-gray-200 p-6;
  }
  
  .btn-primary {
    @apply bg-green-600 hover:bg-green-700 text-white font-medium py-2 px-4 rounded-md transition-colors;
  }
}

/* Mobile optimizations */
@media (max-width: 768px) {
  .mobile-nav {
    @apply fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 px-4 py-2;
  }
  
  .mobile-nav-item {
    @apply flex flex-col items-center justify-center text-xs text-gray-600 hover:text-green-600;
  }
}
```

## 🔄 Real-time Updates

### WebSocket Integration

```typescript
// lib/websocket.ts
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export function useWebSocket(url: string) {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const newSocket = io(url);

    newSocket.on('connect', () => {
      setIsConnected(true);
    });

    newSocket.on('disconnect', () => {
      setIsConnected(false);
    });

    setSocket(newSocket);

    return () => {
      newSocket.close();
    };
  }, [url]);

  return { socket, isConnected };
}

// Usage in components
export function useAgriDAOUpdates(daoId: string) {
  const { socket } = useWebSocket(process.env.NEXT_PUBLIC_WS_URL!);
  const [updates, setUpdates] = useState([]);

  useEffect(() => {
    if (!socket) return;

    socket.emit('join-dao', daoId);
    
    socket.on('dao-update', (update) => {
      setUpdates(prev => [update, ...prev]);
    });

    return () => {
      socket.emit('leave-dao', daoId);
      socket.off('dao-update');
    };
  }, [socket, daoId]);

  return updates;
}
```

---

*Dokumen ini mencakup arsitektur frontend. Untuk styling guidelines dan testing strategies, lihat dokumen terpisah.*
