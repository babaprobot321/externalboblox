using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Runtime.InteropServices;
using System.Threading;
using System.Diagnostics;

namespace test
{
    // ============================================================================
    // NATIVE WIN32 API IMPORTS
    // ============================================================================
    public static class Native
    {
        [DllImport("kernel32.dll")]
        public static extern IntPtr OpenProcess(int dwDesiredAccess, bool bInheritHandle, int dwProcessId);

        [DllImport("kernel32.dll")]
        public static extern bool ReadProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, int dwSize, ref int lpNumberOfBytesRead);

        [DllImport("kernel32.dll")]
        public static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, int dwSize, ref int lpNumberOfBytesWritten);

        [DllImport("kernel32.dll")]
        public static extern bool CloseHandle(IntPtr hObject);

        public const int PROCESS_VM_READ = 0x0010;
        public const int PROCESS_VM_WRITE = 0x0020;
        public const int PROCESS_VM_OPERATION = 0x0008;
    }

    // ============================================================================
    // MEMORY MANAGER - Core reading/writing functionality
    // ============================================================================
    public class MemoryManager
    {
        private IntPtr processHandle;
        private Process process;

        public IntPtr BaseAddress { get; private set; }

        public bool Attach(string processName)
        {
            try
            {
                var processes = Process.GetProcessesByName(processName);
                if (processes.Length == 0)
                    return false;

                process = processes[0];
                processHandle = Native.OpenProcess(
                    Native.PROCESS_VM_READ | Native.PROCESS_VM_WRITE | Native.PROCESS_VM_OPERATION,
                    false,
                    process.Id
                );

                if (processHandle == IntPtr.Zero)
                    return false;

                BaseAddress = process.MainModule.BaseAddress;
                Console.WriteLine($"[SUCCESS] Attached to {processName}");
                Console.WriteLine($"[INFO] Base Address: 0x{BaseAddress.ToInt64():X}");
                Console.WriteLine($"[INFO] Process ID: {process.Id}");
                return true;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[ERROR] Failed to attach: {ex.Message}");
                return false;
            }
        }

        public T Read<T>(IntPtr address) where T : struct
        {
            int bytesRead = 0;
            byte[] buffer = new byte[Marshal.SizeOf(typeof(T))];
            Native.ReadProcessMemory(processHandle, address, buffer, buffer.Length, ref bytesRead);

            GCHandle handle = GCHandle.Alloc(buffer, GCHandleType.Pinned);
            T result = (T)Marshal.PtrToStructure(handle.AddrOfPinnedObject(), typeof(T));
            handle.Free();
            return result;
        }

        public bool Write<T>(IntPtr address, T value) where T : struct
        {
            int bytesWritten = 0;
            byte[] buffer = new byte[Marshal.SizeOf(typeof(T))];

            GCHandle handle = GCHandle.Alloc(buffer, GCHandleType.Pinned);
            Marshal.StructureToPtr(value, handle.AddrOfPinnedObject(), false);
            handle.Free();

            return Native.WriteProcessMemory(processHandle, address, buffer, buffer.Length, ref bytesWritten);
        }

        public byte[] ReadBytes(IntPtr address, int size)
        {
            int bytesRead = 0;
            byte[] buffer = new byte[size];
            Native.ReadProcessMemory(processHandle, address, buffer, size, ref bytesRead);
            return buffer;
        }

        public string ReadString(IntPtr address, int maxLength = 50)
        {
            byte[] buffer = ReadBytes(address, maxLength);
            return Encoding.ASCII.GetString(buffer).Split('\0')[0];
        }

        public IntPtr ReadPointer(IntPtr baseAddress, params int[] offsets)
        {
            IntPtr address = baseAddress;

            foreach (var offset in offsets)
            {
                address = (IntPtr)Read<long>(address);
                if (address == IntPtr.Zero)
                    return IntPtr.Zero;
                address = IntPtr.Add(address, offset);
            }

            return address;
        }

        public void Close()
        {
            if (processHandle != IntPtr.Zero)
                Native.CloseHandle(processHandle);
        }
    }

    // ============================================================================
    // ROBLOX OFFSETS - Update these with Cheat Engine
    // ============================================================================
    public static class Offsets
    {
        // EXAMPLE OFFSETS - YOU MUST FIND CURRENT ONES
        // Use Cheat Engine to locate these addresses

        public static class Pointers
        {
            public const int LocalPlayer = 0x0;      // Base + offset to LocalPlayer
            public const int GameManager = 0x0;       // Base + offset to game manager
        }

        public static class Player
        {
            public const int Character = 0x48;        // LocalPlayer + 0x48 -> Character
            public const int Name = 0x28;             // Player + 0x28 -> Name
        }

        public static class Character
        {
            public const int Humanoid = 0x2E8;        // Character + offset -> Humanoid
            public const int RootPart = 0x2F0;        // Character + offset -> HumanoidRootPart
        }

        public static class Humanoid
        {
            public const int Health = 0x1D0;          // Humanoid + offset -> Health
            public const int MaxHealth = 0x1D8;       // Humanoid + offset -> MaxHealth
            public const int WalkSpeed = 0x48;        // Humanoid + offset -> WalkSpeed
            public const int JumpPower = 0x50;        // Humanoid + offset -> JumpPower
        }

        public static class Part
        {
            public const int Position = 0x11C;        // Part + offset -> Position (Vector3)
            public const int CanCollide = 0x88;       // Part + offset -> CanCollide
        }
    }

    // ============================================================================
    // VECTOR3 STRUCTURE - For positions
    // ============================================================================
    [StructLayout(LayoutKind.Sequential)]
    public struct Vector3
    {
        public float X;
        public float Y;
        public float Z;

        public Vector3(float x, float y, float z)
        {
            X = x;
            Y = y;
            Z = z;
        }

        public override string ToString() => $"({X:F2}, {Y:F2}, {Z:F2})";
    }

    // ============================================================================
    // ROBLOX FEATURES - All cheat functionality
    // ============================================================================
    public class RobloxFeatures
    {
        private MemoryManager mem;
        private bool running = true;

        public RobloxFeatures(MemoryManager memory)
        {
            mem = memory;
        }

        private IntPtr GetLocalPlayer()
        {
            // You need to find this pointer chain with CE
            // Usually: Base -> Some Offset -> LocalPlayer
            IntPtr localPlayerPtr = IntPtr.Add(mem.BaseAddress, Offsets.Pointers.LocalPlayer);
            return mem.Read<IntPtr>(localPlayerPtr);
        }

        private IntPtr GetCharacter()
        {
            IntPtr player = GetLocalPlayer();
            if (player == IntPtr.Zero) return IntPtr.Zero;
            return mem.Read<IntPtr>(IntPtr.Add(player, Offsets.Player.Character));
        }

        private IntPtr GetHumanoid()
        {
            IntPtr character = GetCharacter();
            if (character == IntPtr.Zero) return IntPtr.Zero;
            return mem.Read<IntPtr>(IntPtr.Add(character, Offsets.Character.Humanoid));
        }

        private IntPtr GetRootPart()
        {
            IntPtr character = GetCharacter();
            if (character == IntPtr.Zero) return IntPtr.Zero;
            return mem.Read<IntPtr>(IntPtr.Add(character, Offsets.Character.RootPart));
        }

        public void SetWalkSpeed(float speed)
        {
            IntPtr humanoid = GetHumanoid();
            if (humanoid == IntPtr.Zero)
            {
                Console.WriteLine("[ERROR] Could not find Humanoid");
                return;
            }

            IntPtr speedAddr = IntPtr.Add(humanoid, Offsets.Humanoid.WalkSpeed);
            mem.Write(speedAddr, speed);
            Console.WriteLine($"[SPEED] Set to {speed}");
        }

        public void SetJumpPower(float power)
        {
            IntPtr humanoid = GetHumanoid();
            if (humanoid == IntPtr.Zero)
            {
                Console.WriteLine("[ERROR] Could not find Humanoid");
                return;
            }

            IntPtr jumpAddr = IntPtr.Add(humanoid, Offsets.Humanoid.JumpPower);
            mem.Write(jumpAddr, power);
            Console.WriteLine($"[JUMP] Set to {power}");
        }

        public void SetHealth(float health)
        {
            IntPtr humanoid = GetHumanoid();
            if (humanoid == IntPtr.Zero)
            {
                Console.WriteLine("[ERROR] Could not find Humanoid");
                return;
            }

            IntPtr healthAddr = IntPtr.Add(humanoid, Offsets.Humanoid.Health);
            mem.Write(healthAddr, health);
            Console.WriteLine($"[HEALTH] Set to {health}");
        }

        public float GetHealth()
        {
            IntPtr humanoid = GetHumanoid();
            if (humanoid == IntPtr.Zero) return 0;

            IntPtr healthAddr = IntPtr.Add(humanoid, Offsets.Humanoid.Health);
            return mem.Read<float>(healthAddr);
        }

        public void GodMode(bool enable)
        {
            if (enable)
            {
                Console.WriteLine("[GOD MODE] Enabled - healing loop started");
                new Thread(() =>
                {
                    while (running)
                    {
                        SetHealth(100);
                        Thread.Sleep(100);
                    }
                }).Start();
            }
            else
            {
                Console.WriteLine("[GOD MODE] Disabled");
                running = false;
            }
        }

        public void Teleport(float x, float y, float z)
        {
            IntPtr rootPart = GetRootPart();
            if (rootPart == IntPtr.Zero)
            {
                Console.WriteLine("[ERROR] Could not find RootPart");
                return;
            }

            Vector3 newPos = new Vector3(x, y, z);
            IntPtr posAddr = IntPtr.Add(rootPart, Offsets.Part.Position);
            mem.Write(posAddr, newPos);
            Console.WriteLine($"[TELEPORT] Moved to {newPos}");
        }

        public Vector3 GetPosition()
        {
            IntPtr rootPart = GetRootPart();
            if (rootPart == IntPtr.Zero) return new Vector3(0, 0, 0);

            IntPtr posAddr = IntPtr.Add(rootPart, Offsets.Part.Position);
            return mem.Read<Vector3>(posAddr);
        }

        public void NoClip(bool enable)
        {
            IntPtr rootPart = GetRootPart();
            if (rootPart == IntPtr.Zero)
            {
                Console.WriteLine("[ERROR] Could not find RootPart");
                return;
            }

            IntPtr collideAddr = IntPtr.Add(rootPart, Offsets.Part.CanCollide);
            mem.Write(collideAddr, !enable); // false = no collision
            Console.WriteLine($"[NOCLIP] {(enable ? "Enabled" : "Disabled")}");
        }

        public void Stop()
        {
            running = false;
        }
    }

    // ============================================================================
    // MAIN PROGRAM
    // ============================================================================
    class Program
    {
        static void PrintMenu()
        {
            Console.WriteLine("\n========== ROBLOX EXTERNAL C# ==========");
            Console.WriteLine("[1] Speed Hack");
            Console.WriteLine("[2] Jump Power");
            Console.WriteLine("[3] Set Health");
            Console.WriteLine("[4] God Mode (Auto-heal)");
            Console.WriteLine("[5] Teleport");
            Console.WriteLine("[6] NoClip");
            Console.WriteLine("[7] Show Position");
            Console.WriteLine("[8] Show Current Health");
            Console.WriteLine("[0] Exit");
            Console.WriteLine("=========================================");
            Console.Write("Choice: ");
        }

        static void Main(string[] args)
        {
            Console.Title = "Roblox External - C#";
            Console.WriteLine("ROBLOX EXTERNAL CHEAT (C#)");
            Console.WriteLine("Waiting for Roblox...\n");

            MemoryManager mem = new MemoryManager();

            // Wait for Roblox to start
            while (!mem.Attach("RobloxPlayerBeta"))
            {
                Console.WriteLine("Waiting for RobloxPlayerBeta.exe...");
                Thread.Sleep(2000);
            }

            RobloxFeatures features = new RobloxFeatures(mem);
            bool running = true;
            bool godModeActive = false;

            Console.WriteLine("\nPress any key to open menu...");
            Console.ReadKey();

            while (running)
            {
                Console.Clear();
                PrintMenu();

                string input = Console.ReadLine();
                if (!int.TryParse(input, out int choice))
                    continue;

                switch (choice)
                {
                    case 1:
                        Console.Write("Enter speed (16 default, 100 fast): ");
                        if (float.TryParse(Console.ReadLine(), out float speed))
                            features.SetWalkSpeed(speed);
                        break;

                    case 2:
                        Console.Write("Enter jump power (50 default, 200 high): ");
                        if (float.TryParse(Console.ReadLine(), out float jump))
                            features.SetJumpPower(jump);
                        break;

                    case 3:
                        Console.Write("Enter health (0-100): ");
                        if (float.TryParse(Console.ReadLine(), out float health))
                            features.SetHealth(health);
                        break;

                    case 4:
                        godModeActive = !godModeActive;
                        features.GodMode(godModeActive);
                        Console.WriteLine("Press any key to continue...");
                        Console.ReadKey();
                        break;

                    case 5:
                        Console.Write("Enter X: ");
                        float.TryParse(Console.ReadLine(), out float x);
                        Console.Write("Enter Y: ");
                        float.TryParse(Console.ReadLine(), out float y);
                        Console.Write("Enter Z: ");
                        float.TryParse(Console.ReadLine(), out float z);
                        features.Teleport(x, y, z);
                        break;

                    case 6:
                        Console.Write("Enable NoClip? (y/n): ");
                        bool noclip = Console.ReadLine().ToLower() == "y";
                        features.NoClip(noclip);
                        break;

                    case 7:
                        Vector3 pos = features.GetPosition();
                        Console.WriteLine($"Current position: {pos}");
                        Console.WriteLine("Press any key to continue...");
                        Console.ReadKey();
                        break;

                    case 8:
                        float currentHealth = features.GetHealth();
                        Console.WriteLine($"Current health: {currentHealth}");
                        Console.WriteLine("Press any key to continue...");
                        Console.ReadKey();
                        break;

                    case 0:
                        running = false;
                        features.Stop();
                        break;

                    default:
                        Console.WriteLine("Invalid choice");
                        Thread.Sleep(1000);
                        break;
                }
            }

            mem.Close();
            Console.WriteLine("\nExiting...");
        }
    }
}
