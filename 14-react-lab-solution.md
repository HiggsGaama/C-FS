<!-- 14-react-lab-solution.md -->
# 3.5 Module 3 Hands-On Lab: Interactive Product & Stock Dashboard

## Lab Scenario
You will build a responsive, production-ready React 18 + TypeScript web application connecting to a backend REST API. The application includes a searchable inventory grid, debounced search filtering, a modal form with React Hook Form and Zod validation, and TanStack Query for cache invalidation and mutation handling.

---

## Lab Requirements
1. **Setup Vite + React + TypeScript + Tailwind CSS**.
2. **State & Networking**:
   - `TanStack Query v5` (`useQuery`, `useMutation`).
   - Mock API layer simulating network latency and HTTP responses.
3. **Components**:
   - `InventoryDashboard`: Main container with metric cards.
   - `ProductTable`: Responsive table with stock status badges.
   - `ProductModal`: Controlled modal with accessible focus management.
   - `CreateProductForm`: Form validated via Zod and React Hook Form.

---

## Complete Solution Code

### 1. Project Initialization
```bash
npm create vite@latest module03-react-dashboard -- --template react-ts
cd module03-react-dashboard

npm install @tanstack/react-query @hookform/resolvers react-hook-form zod axios
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 2. Complete Application Implementation (`src/App.tsx`)

Replace `src/App.tsx` with the following comprehensive, self-contained implementation:

```tsx
import React, { useState, useMemo, useEffect } from 'react';
import {
  QueryClient,
  QueryClientProvider,
  useQuery,
  useMutation,
  useQueryClient,
} from '@tanstack/react-query';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// ============================================================================
// 1. SCHEMAS & TYPES
// ============================================================================

export interface ProductItem {
  id: string;
  sku: string;
  name: string;
  category: string;
  unitPrice: number;
  stockQuantity: number;
}

const productSchema = z.object({
  sku: z.string().min(4, 'SKU must be at least 4 characters').regex(/^[A-Z0-9-]+$/, 'Alphanumeric & dashes only'),
  name: z.string().min(3, 'Name is required').max(80),
  category: z.string().min(1, 'Category is required'),
  unitPrice: z.coerce.number().positive('Price must be greater than zero'),
  stockQuantity: z.coerce.number().int().min(0, 'Stock cannot be negative'),
});

type ProductFormData = z.infer<typeof productSchema>;

// ============================================================================
// 2. MOCK API CLIENT (Simulates Network Latency & Persistence)
// ============================================================================

let MOCK_DATABASE: ProductItem[] = [
  { id: '1', sku: 'LAPTOP-PRO-16', name: 'MacBook Pro 16" M3 Max', category: 'Laptops', unitPrice: 3499.0, stockQuantity: 8 },
  { id: '2', sku: 'DISP-4K-DELL', name: 'Dell UltraSharp 32" 4K', category: 'Displays', unitPrice: 899.99, stockQuantity: 15 },
  { id: '3', sku: 'KEY-MECH-MX', name: 'Logitech MX Mechanical Keyboard', category: 'Accessories', unitPrice: 169.99, stockQuantity: 42 },
  { id: '4', sku: 'MOU-ERGO-V3', name: 'Logitech MX Master 3S Mouse', category: 'Accessories', unitPrice: 99.99, stockQuantity: 0 },
];

const mockApi = {
  fetchProducts: async (): Promise<ProductItem[]> => {
    await new Promise((resolve) => setTimeout(resolve, 400));
    return [...MOCK_DATABASE];
  },
  createProduct: async (input: ProductFormData): Promise<ProductItem> => {
    await new Promise((resolve) => setTimeout(resolve, 600));
    const newItem: ProductItem = {
      id: String(Date.now()),
      sku: input.sku.toUpperCase(),
      name: input.name,
      category: input.category,
      unitPrice: input.unitPrice,
      stockQuantity: input.stockQuantity,
    };
    MOCK_DATABASE = [newItem, ...MOCK_DATABASE];
    return newItem;
  },
};

// ============================================================================
// 3. HOOKS
// ============================================================================

function useDebounce<T>(value: T, delayMs: number = 300): T {
  const [debounced, setDebounced] = useState<T>(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(timer);
  }, [value, delayMs]);
  return debounced;
}

// ============================================================================
// 4. SUBCOMPONENTS
// ============================================================================

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

function Modal({ isOpen, onClose, title, children }: ModalProps) {
  if (!isOpen) return null;
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-black/50 backdrop-blur-sm">
      <div className="bg-white rounded-xl shadow-2xl max-w-md w-full p-6 relative">
        <div className="flex justify-between items-center pb-3 border-b border-slate-100">
          <h3 className="text-lg font-bold text-slate-800">{title}</h3>
          <button
            type="button"
            onClick={onClose}
            className="text-slate-400 hover:text-slate-700 text-xl font-bold"
          >
            &times;
          </button>
        </div>
        <div className="mt-4">{children}</div>
      </div>
    </div>
  );
}

// ============================================================================
// 5. MAIN DASHBOARD COMPONENT
// ============================================================================

function Dashboard() {
  const queryClient = useQueryClient();
  const [search, setSearch] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('ALL');
  const [isModalOpen, setIsModalOpen] = useState(false);

  const debouncedSearch = useDebounce(search, 250);

  // Fetch products query
  const { data: products = [], isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: mockApi.fetchProducts,
  });

  // Create product mutation
  const createMutation = useMutation({
    mutationFn: mockApi.createProduct,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
      setIsModalOpen(false);
    },
  });

  // Filtered dataset
  const filteredProducts = useMemo(() => {
    return products.filter((item) => {
      const matchesSearch =
        item.name.toLowerCase().includes(debouncedSearch.toLowerCase()) ||
        item.sku.toLowerCase().includes(debouncedSearch.toLowerCase());
      const matchesCategory = selectedCategory === 'ALL' || item.category === selectedCategory;
      return matchesSearch && matchesCategory;
    });
  }, [products, debouncedSearch, selectedCategory]);

  const totalValue = useMemo(() => {
    return products.reduce((acc, p) => acc + p.unitPrice * p.stockQuantity, 0);
  }, [products]);

  // Form setup
  const {
    register,
    handleSubmit,
    reset,
    formState: { errors },
  } = useForm<ProductFormData>({
    resolver: zodResolver(productSchema),
    defaultValues: { sku: '', name: '', category: 'Accessories', unitPrice: 0, stockQuantity: 0 },
  });

  const onFormSubmit = (data: ProductFormData) => {
    createMutation.mutate(data, {
      onSuccess: () => reset(),
    });
  };

  return (
    <div className="min-h-screen bg-slate-50 p-6 md:p-10 font-sans">
      <div className="max-w-6xl mx-auto space-y-6">
        {/* Header */}
        <div className="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4">
          <div>
            <h1 className="text-3xl font-extrabold text-slate-900">Inventory Dashboard</h1>
            <p className="text-sm text-slate-500 mt-1">
              Real-time enterprise warehouse stock tracking & mutation pipeline
            </p>
          </div>
          <button
            type="button"
            onClick={() => setIsModalOpen(true)}
            className="px-4 py-2 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold text-sm rounded-lg shadow-sm transition"
          >
            + Add New Product
          </button>
        </div>

        {/* Metrics Bar */}
        <div className="grid grid-cols-1 sm:grid-cols-3 gap-4">
          <div className="p-4 bg-white border border-slate-200 rounded-lg shadow-xs">
            <span className="text-xs font-semibold text-slate-500 uppercase">Total SKUs</span>
            <p className="text-2xl font-bold text-slate-800 mt-1">{products.length}</p>
          </div>
          <div className="p-4 bg-white border border-slate-200 rounded-lg shadow-xs">
            <span className="text-xs font-semibold text-slate-500 uppercase">Total Inventory Valuation</span>
            <p className="text-2xl font-bold text-emerald-600 mt-1">
              {totalValue.toLocaleString('en-US', { style: 'currency', currency: 'USD' })}
            </p>
          </div>
          <div className="p-4 bg-white border border-slate-200 rounded-lg shadow-xs">
            <span className="text-xs font-semibold text-slate-500 uppercase">Out of Stock Alerts</span>
            <p className="text-2xl font-bold text-rose-600 mt-1">
              {products.filter((p) => p.stockQuantity === 0).length} items
            </p>
          </div>
        </div>

        {/* Filters */}
        <div className="flex flex-col sm:flex-row gap-3 bg-white p-4 rounded-lg border border-slate-200">
          <input
            type="text"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Search by SKU or title..."
            className="flex-1 px-3 py-2 border border-slate-300 rounded-md text-sm focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          />
          <select
            value={selectedCategory}
            onChange={(e) => setSelectedCategory(e.target.value)}
            className="px-3 py-2 border border-slate-300 rounded-md text-sm focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          >
            <option value="ALL">All Categories</option>
            <option value="Laptops">Laptops</option>
            <option value="Displays">Displays</option>
            <option value="Accessories">Accessories</option>
          </select>
        </div>

        {/* Data Table */}
        <div className="bg-white border border-slate-200 rounded-lg shadow-xs overflow-x-auto">
          {isLoading ? (
            <div className="p-12 text-center text-slate-500 font-medium">Loading inventory data...</div>
          ) : error ? (
            <div className="p-12 text-center text-rose-600 font-medium">Error loading data.</div>
          ) : filteredProducts.length === 0 ? (
            <div className="p-12 text-center text-slate-400 font-medium">No matching items found.</div>
          ) : (
            <table className="w-full text-left border-collapse">
              <thead>
                <tr className="border-b border-slate-200 bg-slate-50 text-[11px] font-bold uppercase text-slate-500 tracking-wider">
                  <th className="py-3 px-4">SKU</th>
                  <th className="py-3 px-4">Product Name</th>
                  <th className="py-3 px-4">Category</th>
                  <th className="py-3 px-4 text-right">Price</th>
                  <th className="py-3 px-4 text-right">Stock</th>
                  <th className="py-3 px-4 text-center">Status</th>
                </tr>
              </thead>
              <tbody className="divide-y divide-slate-100 text-sm">
                {filteredProducts.map((p) => {
                  const isOut = p.stockQuantity === 0;
                  const isLow = p.stockQuantity > 0 && p.stockQuantity <= 10;
                  return (
                    <tr key={p.id} className="hover:bg-slate-50 transition">
                      <td className="py-3 px-4 font-mono font-medium text-slate-600">{p.sku}</td>
                      <td className="py-3 px-4 font-semibold text-slate-800">{p.name}</td>
                      <td className="py-3 px-4 text-slate-500">{p.category}</td>
                      <td className="py-3 px-4 text-right font-medium text-slate-800">
                        {p.unitPrice.toLocaleString('en-US', { style: 'currency', currency: 'USD' })}
                      </td>
                      <td className="py-3 px-4 text-right font-bold">{p.stockQuantity}</td>
                      <td className="py-3 px-4 text-center">
                        <span
                          className={`inline-block px-2.5 py-0.5 rounded-full text-xs font-semibold ${
                            isOut
                              ? 'bg-rose-100 text-rose-800'
                              : isLow
                              ? 'bg-amber-100 text-amber-800'
                              : 'bg-emerald-100 text-emerald-800'
                          }`}
                        >
                          {isOut ? 'Out of Stock' : isLow ? 'Low Stock' : 'In Stock'}
                        </span>
                      </td>
                    </tr>
                  );
                })}
              </tbody>
            </table>
          )}
        </div>

        {/* Modal for Creating Product */}
        <Modal isOpen={isModalOpen} onClose={() => setIsModalOpen(false)} title="Add Product to Stock">
          <form onSubmit={handleSubmit(onFormSubmit)} className="space-y-4">
            <div>
              <label htmlFor="modal-sku" className="block text-xs font-semibold text-slate-700 uppercase mb-1">
                SKU
              </label>
              <input
                id="modal-sku"
                {...register('sku')}
                placeholder="SKU-8821"
                className="w-full px-3 py-2 border rounded-md text-sm border-slate-300 focus:ring-2 focus:ring-indigo-500"
              />
              {errors.sku && <p className="text-xs text-rose-600 mt-1">{errors.sku.message}</p>}
            </div>

            <div>
              <label htmlFor="modal-name" className="block text-xs font-semibold text-slate-700 uppercase mb-1">
                Product Name
              </label>
              <input
                id="modal-name"
                {...register('name')}
                placeholder="Ultra Slim Monitor Stand"
                className="w-full px-3 py-2 border rounded-md text-sm border-slate-300 focus:ring-2 focus:ring-indigo-500"
              />
              {errors.name && <p className="text-xs text-rose-600 mt-1">{errors.name.message}</p>}
            </div>

            <div>
              <label htmlFor="modal-cat" className="block text-xs font-semibold text-slate-700 uppercase mb-1">
                Category
              </label>
              <select
                id="modal-cat"
                {...register('category')}
                className="w-full px-3 py-2 border rounded-md text-sm border-slate-300 focus:ring-2 focus:ring-indigo-500"
              >
                <option value="Laptops">Laptops</option>
                <option value="Displays">Displays</option>
                <option value="Accessories">Accessories</option>
              </select>
            </div>

            <div className="grid grid-cols-2 gap-3">
              <div>
                <label htmlFor="modal-price" className="block text-xs font-semibold text-slate-700 uppercase mb-1">
                  Price ($)
                </label>
                <input
                  id="modal-price"
                  type="number"
                  step="0.01"
                  {...register('unitPrice')}
                  className="w-full px-3 py-2 border rounded-md text-sm border-slate-300 focus:ring-2 focus:ring-indigo-500"
                />
                {errors.unitPrice && (
                  <p className="text-xs text-rose-600 mt-1">{errors.unitPrice.message}</p>
                )}
              </div>
              <div>
                <label htmlFor="modal-qty" className="block text-xs font-semibold text-slate-700 uppercase mb-1">
                  Quantity
                </label>
                <input
                  id="modal-qty"
                  type="number"
                  {...register('stockQuantity')}
                  className="w-full px-3 py-2 border rounded-md text-sm border-slate-300 focus:ring-2 focus:ring-indigo-500"
                />
                {errors.stockQuantity && (
                  <p className="text-xs text-rose-600 mt-1">{errors.stockQuantity.message}</p>
                )}
              </div>
            </div>

            <div className="pt-2 flex justify-end gap-2">
              <button
                type="button"
                onClick={() => setIsModalOpen(false)}
                className="px-4 py-2 border border-slate-300 text-slate-700 rounded-md text-sm font-medium hover:bg-slate-50"
              >
                Cancel
              </button>
              <button
                type="submit"
                disabled={createMutation.isPending}
                className="px-4 py-2 bg-indigo-600 hover:bg-indigo-700 disabled:opacity-50 text-white rounded-md text-sm font-medium transition"
              >
                {createMutation.isPending ? 'Saving...' : 'Save Product'}
              </button>
            </div>
          </form>
        </Modal>
      </div>
    </div>
  );
}

// ============================================================================
// 6. ROOT APP WITH QUERY CLIENT PROVIDER
// ============================================================================

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60, // 1 minute
      refetchOnWindowFocus: false,
    },
  },
});

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Dashboard />
    </QueryClientProvider>
  );
}
```

---

## Verification & Execution Steps

1. Start the Vite development server:
   ```bash
   npm run dev
   ```
2. Open `http://localhost:5173` in your browser.
3. Verify live features:
   - Debounced search across SKUs and names.
   - Category dropdown filtering.
   - Click "+ Add New Product" to launch the modal.
   - Attempt submitting with empty or invalid fields to verify Zod inline error messages.
   - Submit valid data; observe modal closing and immediate optimistic table update.
