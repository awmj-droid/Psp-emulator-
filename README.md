# Psp-emulator-
Put games in 
import struct
from enum import Enum
from typing import List, Dict
import pygame

class CPUState:
    """Represents the MIPS CPU state"""
    def __init__(self):
        # 32 general-purpose registers
        self.registers = [0] * 32
        # Program counter
        self.pc = 0
        # Hi/Lo registers for multiplication/division
        self.hi = 0
        self.lo = 0

class Memory:
    """PSP Memory Management"""
    def __init__(self):
        # PSP has 32MB of main RAM
        self.ram = bytearray(32 * 1024 * 1024)
        # Scratchpad RAM (fast memory)
        self.scratchpad = bytearray(64 * 1024)
    
    def read_byte(self, address: int) -> int:
        if address < len(self.ram):
            return self.ram[address]
        return 0
    
    def read_word(self, address: int) -> int:
        """Read 32-bit word"""
        return struct.unpack('<I', self.ram[address:address+4])[0]
    
    def write_byte(self, address: int, value: int) -> None:
        if address < len(self.ram):
            self.ram[address] = value & 0xFF
    
    def write_word(self, address: int, value: int) -> None:
        """Write 32-bit word"""
        self.ram[address:address+4] = struct.pack('<I', value & 0xFFFFFFFF)

class MIPSCPUEmulator:
    """MIPS CPU Emulator for PSP"""
    def __init__(self, memory: Memory):
        self.cpu = CPUState()
        self.memory = memory
    
    def fetch_instruction(self) -> int:
        """Fetch 32-bit instruction from current PC"""
        instruction = self.memory.read_word(self.cpu.pc)
        self.cpu.pc += 4
        return instruction
    
    def decode_instruction(self, instr: int) -> Dict:
        """Decode MIPS instruction"""
        opcode = (instr >> 26) & 0x3F
        rs = (instr >> 21) & 0x1F
        rt = (instr >> 16) & 0x1F
        rd = (instr >> 11) & 0x1F
        immediate = instr & 0xFFFF
        
        return {
            'opcode': opcode,
            'rs': rs,
            'rt': rt,
            'rd': rd,
            'immediate': immediate,
            'raw': instr
        }
    
    def execute_instruction(self, instr_data: Dict) -> None:
        """Execute decoded instruction"""
        opcode = instr_data['opcode']
        
        # ADD instruction (opcode 0x00, funct 0x20)
        if opcode == 0:
            funct = instr_data['raw'] & 0x3F
            if funct == 0x20:  # ADD
                self.cpu.registers[instr_data['rd']] = (
                    self.cpu.registers[instr_data['rs']] +
                    self.cpu.registers[instr_data['rt']]
                ) & 0xFFFFFFFF
        
        # ADDI instruction (opcode 0x08)
        elif opcode == 0x08:  # ADDI
            self.cpu.registers[instr_data['rt']] = (
                self.cpu.registers[instr_data['rs']] +
                instr_data['immediate']
            ) & 0xFFFFFFFF
        
        # Add more instructions as needed
    
    def step(self) -> None:
        """Execute one CPU cycle"""
        instruction = self.fetch_instruction()
        instr_data = self.decode_instruction(instruction)
        self.execute_instruction(instr_data)

class PSPEmulator:
    """Main PSP Emulator"""
    def __init__(self, rom_path: str):
        self.memory = Memory()
        self.cpu = MIPSCPUEmulator(self.memory)
        self.rom_path = rom_path
        self.running = False
        
        # Initialize pygame for graphics
        pygame.init()
        self.screen = pygame.display.set_mode((480, 272))  # PSP screen resolution
        pygame.display.set_caption("PSP Emulator")
    
    def load_rom(self, path: str) -> bool:
        """Load PSP ISO/ROM file"""
        try:
            with open(path, 'rb') as f:
                rom_data = f.read()
            # Copy ROM to memory (simplified)
            self.memory.ram[:len(rom_data)] = rom_data
            return True
        except Exception as e:
            print(f"Error loading ROM: {e}")
            return False
    
    def run(self) -> None:
        """Main emulator loop"""
        if not self.load_rom(self.rom_path):
            return
        
        self.running = True
        clock = pygame.time.Clock()
        
        while self.running:
            # Handle events
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False
            
            # Execute CPU cycles
            for _ in range(100000):  # Execute multiple cycles per frame
                self.cpu.step()
            
            # Render (placeholder)
            self.screen.fill((0, 0, 0))
            pygame.display.flip()
            clock.tick(60)  # 60 FPS
        
        pygame.quit()

if __name__ == "__main__":
    emulator = PSPEmulator("game.iso")
    emulator.run()
