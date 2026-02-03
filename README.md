"""
Urban City Parking Management System

Name: MD SAHADAT HOSSAIN MARUF
Student Id: 68192


"""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta
from abc import ABC, abstractmethod
from typing import Dict, List, Optional, Tuple, Any
import uuid



# Domain Models: Vehicle


@dataclass
class Vehicle:
    plate_number: str
    vehicle_type: str
    entry_time: Optional[datetime] = None
    exit_time: Optional[datetime] = None

    def set_entry_time(self, when: Optional[datetime] = None) -> None:
        self.entry_time = when or datetime.now()

    def set_exit_time(self, when: Optional[datetime] = None) -> None:
        self.exit_time = when or datetime.now()

    def calculate_parking_duration_hours(self) -> float:
        """
        Returns duration in hours (float). If exit_time is None, uses current time.
        """
        if self.entry_time is None:
            return 0.0
        end = self.exit_time or datetime.now()
        delta = end - self.entry_time
        return max(delta.total_seconds() / 3600.0, 0.0)

    def get_vehicle_info(self) -> Dict[str, Any]:
        return {
            "plate_number": self.plate_number,
            "vehicle_type": self.vehicle_type,
            "entry_time": self.entry_time.isoformat() if self.entry_time else None,
            "exit_time": self.exit_time.isoformat() if self.exit_time else None,
        }


@dataclass
class Car(Vehicle):
    passenger_capacity: int = 5
    is_electric: bool = False

    def __post_init__(self) -> None:
        self.vehicle_type = "Car"


@dataclass
class Motorcycle(Vehicle):
    engine_cc: int = 150
    has_top_box: bool = False

    def __post_init__(self) -> None:
        self.vehicle_type = "Motorcycle"


# Domain Models: Parking Space


@dataclass
class ParkingSpace:
    space_id: int
    space_type: str
    location_zone: str
    is_occupied: bool = False
    current_vehicle: Optional[Vehicle] = None

    def occupy_space(self, vehicle: Vehicle) -> bool:
        if self.is_occupied:
            return False
        self.is_occupied = True
        self.current_vehicle = vehicle
        return True

    def vacate_space(self) -> Optional[Vehicle]:
        vehicle = self.current_vehicle
        self.is_occupied = False
        self.current_vehicle = None
        return vehicle

    def is_available(self) -> bool:
        return not self.is_occupied

    def get_space_status(self) -> Dict[str, Any]:
        return {
            "space_id": self.space_id,
            "space_type": self.space_type,
            "location_zone": self.location_zone,
            "is_occupied": self.is_occupied,
            "current_vehicle": self.current_vehicle.plate_number if self.current_vehicle else None,
        }



# Domain Models: Ticket


@dataclass
class ParkingTicket:
    ticket_id: str
    vehicle_plate: str
    entry_time: datetime
    space_id: int

    def generate_ticket(self) -> str:
        return f"TICKET {self.ticket_id} | Plate: {self.vehicle_plate} | Space: {self.space_id} | Entry: {self.entry_time.isoformat()}"

    def get_ticket_details(self) -> Dict[str, Any]:
        return {
            "ticket_id": self.ticket_id,
            "vehicle_plate": self.vehicle_plate,
            "entry_time": self.entry_time.isoformat(),
            "space_id": self.space_id,
        }

    @staticmethod
    def calculate_expected_fee(hourly_rate: float, duration_hours: float) -> float:
        return round(max(duration_hours, 0.0) * hourly_rate, 2)



# Payment Strategy (ABC)


@dataclass
class Payment(ABC):
    payment_id: str
    timestamp: datetime = field(default_factory=datetime.now)
    receipt_number: str = field(default_factory=lambda: uuid.uuid4().hex[:10].upper())

    @abstractmethod
    def calculate_fee(self, duration_hours: float, context: Dict[str, Any]) -> float:
        """Calculate fee for the provided duration and context."""
        raise NotImplementedError

    @abstractmethod
    def process_payment(self, amount: float) -> bool:
        """Process payment; returns True on success."""
        raise NotImplementedError

    def generate_receipt(self, amount: float, status: bool) -> str:
        paid_status = "PAID" if status else "FAILED"
        return (
            f"RECEIPT {self.receipt_number} | PaymentID: {self.payment_id} | "
            f"Time: {self.timestamp.isoformat()} | Amount: ${amount:.2f} | Status: {paid_status}"
        )


@dataclass
class HourlyPayment(Payment):
    hourly_rate: float = 5.00
    peak_hour_surcharge: float = 1.50  # multiplier (e.g., 1.5x)

    def apply_peak_hour_surcharge(self, base_fee: float) -> float:
        return round(base_fee * self.peak_hour_surcharge, 2)

    def calculate_fee(self, duration_hours: float, context: Dict[str, Any]) -> float:
        """
        Pricing strategy:
        - Base: hourly_rate * ceil(duration_hours to nearest 0.25 hour)
        - Vehicle type multiplier: Car=1.0, Motorcycle=0.8 (example policy)
        - Peak hours multiplier (e.g., 1.5x)
        - Weekend multiplier (e.g., 1.2x)
        """
        # Round up to nearest 0.25 hour to avoid undercharging for partial time.
        rounded_hours = max(duration_hours, 0.0)
        rounded_hours = (int(rounded_hours * 4 + (0 if rounded_hours * 4 == int(rounded_hours * 4) else 1))) / 4.0

        vehicle_type = (context.get("vehicle_type") or "").lower()
        is_peak = bool(context.get("is_peak", False))
        is_weekend = bool(context.get("is_weekend", False))

        vehicle_multiplier = 1.0
        if vehicle_type == "motorcycle":
            vehicle_multiplier = 0.8
        elif vehicle_type == "car":
            vehicle_multiplier = 1.0

        fee = self.hourly_rate * rounded_hours * vehicle_multiplier

        if is_weekend:
            fee *= 1.2

        fee = round(fee, 2)

        if is_peak:
            fee = self.apply_peak_hour_surcharge(fee)

        return round(fee, 2)

    def process_payment(self, amount: float) -> bool:
        # In a real system, integrate with a payment gateway.
        # For assessment/demo, assume positive amounts always succeed.
        return amount >= 0.0


@dataclass
class MonthlyPayment(Payment):
    monthly_fee: float = 200.00
    valid_from: datetime = field(default_factory=datetime.now)
    valid_to: datetime = field(default_factory=lambda: datetime.now() + timedelta(days=30))
    vehicle_plate: str = ""

    def is_valid(self, on_time: Optional[datetime] = None) -> bool:
        t = on_time or datetime.now()
        return self.valid_from <= t <= self.valid_to

    def renew_subscription(self, days: int = 30) -> None:
        now = datetime.now()
        # Renew from max(current valid_to, now) to avoid losing remaining days.
        start = self.valid_to if self.valid_to > now else now
        self.valid_from = start
        self.valid_to = start + timedelta(days=days)

    def calculate_fee(self, duration_hours: float, context: Dict[str, Any]) -> float:
        # Monthly pass holders pay $0 per visit while pass is valid.
        return 0.0

    def process_payment(self, amount: float) -> bool:
        # For the monthly pass purchase/renewal. Positive amounts assumed success.
        return amount >= 0.0



# Parking System Controller


class ParkingSystem:
    """
    Main controller class managing 300 spaces by default.
    """

    def __init__(self, total_spaces: int = 300) -> None:
        self.total_spaces = total_spaces
        self.available_spaces: List[ParkingSpace] = []
        self.occupied_spaces: Dict[int, ParkingSpace] = {}
        self.parking_records: List[Dict[str, Any]] = []
        self.monthly_pass_holders: Dict[str, MonthlyPayment] = {}  # plate -> MonthlyPayment
        self.active_tickets: Dict[str, ParkingTicket] = {}  # plate -> ticket
        self.active_vehicles: Dict[str, Vehicle] = {}  # plate -> vehicle

        # Initialize spaces: simple zoning policy (A/B/C) and types.
        for i in range(1, total_spaces + 1):
            zone = "A" if i <= 100 else ("B" if i <= 200 else "C")
            space_type = "Standard"
            self.available_spaces.append(ParkingSpace(space_id=i, space_type=space_type, location_zone=zone))

    # ---- Pricing helpers ----

    @staticmethod
    def _is_weekend(t: datetime) -> bool:
        return t.weekday() >= 5  # 5=Sat, 6=Sun

    @staticmethod
    def _is_peak(t: datetime) -> bool:
        # Example peak windows: 8-10 and 16-19
        return (8 <= t.hour < 10) or (16 <= t.hour < 19)

    # ---- Monthly pass ----

    def add_monthly_pass(self, plate_number: str, days: int = 30, monthly_fee: float = 200.0) -> Dict[str, Any]:
        """
        Creates or renews a monthly pass for the given plate number.
        """
        plate = plate_number.strip().upper()
        if plate in self.monthly_pass_holders:
            pass_obj = self.monthly_pass_holders[plate]
            pass_obj.renew_subscription(days=days)
            amount = monthly_fee
            ok = pass_obj.process_payment(amount)
            receipt = pass_obj.generate_receipt(amount, ok)
            return {
                "status": "renewed",
                "plate_number": plate,
                "valid_from": pass_obj.valid_from.isoformat(),
                "valid_to": pass_obj.valid_to.isoformat(),
                "receipt": receipt,
            }

        pass_obj = MonthlyPayment(
            payment_id=uuid.uuid4().hex[:12].upper(),
            monthly_fee=monthly_fee,
            valid_from=datetime.now(),
            valid_to=datetime.now() + timedelta(days=days),
            vehicle_plate=plate,
        )
        amount = monthly_fee
        ok = pass_obj.process_payment(amount)
        receipt = pass_obj.generate_receipt(amount, ok)
        self.monthly_pass_holders[plate] = pass_obj
        return {
            "status": "created",
            "plate_number": plate,
            "valid_from": pass_obj.valid_from.isoformat(),
            "valid_to": pass_obj.valid_to.isoformat(),
            "receipt": receipt,
        }

    def validate_monthly_pass(self, plate_number: str) -> bool:
        plate = plate_number.strip().upper()
        pass_obj = self.monthly_pass_holders.get(plate)
        return bool(pass_obj and pass_obj.is_valid())

    # ---- Space allocation ----

    def find_available_space(self, vehicle_type: str) -> Optional[ParkingSpace]:
        # For this assessment, we allocate first available standard space.
        for space in self.available_spaces:
            if space.is_available():
                return space
        return None

    # ---- Core operations ----

    def vehicle_entry(self, plate_number: str, vehicle_type: str) -> Dict[str, Any]:
        plate = plate_number.strip().upper()
        vtype = vehicle_type.strip().title()

        if plate in self.active_vehicles:
            return {"success": False, "message": f"Vehicle {plate} is already parked."}

        space = self.find_available_space(vtype)
        if not space:
            return {"success": False, "message": "No available parking spaces."}

        # Create vehicle object
        if vtype.lower() == "motorcycle":
            vehicle: Vehicle = Motorcycle(plate_number=plate, vehicle_type="Motorcycle")
        else:
            vehicle = Car(plate_number=plate, vehicle_type="Car")

        vehicle.set_entry_time(datetime.now())

        ok = space.occupy_space(vehicle)
        if not ok:
            return {"success": False, "message": "Selected space could not be occupied."}

        # Move space from available list to occupied dict
        self.available_spaces.remove(space)
        self.occupied_spaces[space.space_id] = space
        self.active_vehicles[plate] = vehicle

        # Ticket for single-entry tracking
        ticket = ParkingTicket(
            ticket_id=uuid.uuid4().hex[:10].upper(),
            vehicle_plate=plate,
            entry_time=vehicle.entry_time,
            space_id=space.space_id,
        )
        self.active_tickets[plate] = ticket

        record = {
            "event": "entry",
            "plate_number": plate,
            "vehicle_type": vehicle.vehicle_type,
            "space_id": space.space_id,
            "timestamp": vehicle.entry_time.isoformat(),
            "ticket_id": ticket.ticket_id,
            "monthly_pass_valid": self.validate_monthly_pass(plate),
        }
        self.parking_records.append(record)

        return {
            "success": True,
            "message": f"Vehicle {plate} parked successfully.",
            "space_id": space.space_id,
            "zone": space.location_zone,
            "entry_time": vehicle.entry_time.isoformat(),
            "ticket": ticket.get_ticket_details(),
            "monthly_pass_valid": record["monthly_pass_valid"],
        }

    def calculate_parking_fee(self, plate_number: str) -> float:
        plate = plate_number.strip().upper()
        vehicle = self.active_vehicles.get(plate)
        if not vehicle:
            return 0.0

        # If monthly pass valid, fee is 0
        if self.validate_monthly_pass(plate):
            return 0.0

        duration = vehicle.calculate_parking_duration_hours()
        now = vehicle.exit_time or datetime.now()
        context = {
            "vehicle_type": vehicle.vehicle_type,
            "is_peak": self._is_peak(now),
            "is_weekend": self._is_weekend(now),
        }
        payment = HourlyPayment(payment_id=uuid.uuid4().hex[:12].upper())
        return payment.calculate_fee(duration, context)

    def vehicle_exit(self, plate_number: str) -> Dict[str, Any]:
        plate = plate_number.strip().upper()
        vehicle = self.active_vehicles.get(plate)
        if not vehicle:
            return {"success": False, "message": f"Vehicle {plate} is not currently parked."}

        vehicle.set_exit_time(datetime.now())
        duration = vehicle.calculate_parking_duration_hours()

        # Find the occupied space for this vehicle
        space_id = None
        for sid, space in self.occupied_spaces.items():
            if space.current_vehicle and space.current_vehicle.plate_number == plate:
                space_id = sid
                break

        if space_id is None:
            return {"success": False, "message": "Internal error: occupied space not found for vehicle."}

        # Calculate fee and choose payment strategy
        monthly_valid = self.validate_monthly_pass(plate)
        if monthly_valid:
            payment: Payment = MonthlyPayment(payment_id=uuid.uuid4().hex[:12].upper(), vehicle_plate=plate)
            amount = 0.0
            ok = True
        else:
            now = vehicle.exit_time
            context = {
                "vehicle_type": vehicle.vehicle_type,
                "is_peak": self._is_peak(now),
                "is_weekend": self._is_weekend(now),
            }
            payment = HourlyPayment(payment_id=uuid.uuid4().hex[:12].upper())
            amount = payment.calculate_fee(duration, context)
            ok = payment.process_payment(amount)

        receipt = payment.generate_receipt(amount, ok)

        # Vacate the space
        space = self.occupied_spaces.pop(space_id)
        space.vacate_space()
        self.available_spaces.append(space)

        # Remove active tracking
        ticket = self.active_tickets.pop(plate, None)
        self.active_vehicles.pop(plate, None)

        record = {
            "event": "exit",
            "plate_number": plate,
            "vehicle_type": vehicle.vehicle_type,
            "space_id": space_id,
            "entry_time": vehicle.entry_time.isoformat() if vehicle.entry_time else None,
            "exit_time": vehicle.exit_time.isoformat() if vehicle.exit_time else None,
            "duration_hours": round(duration, 2),
            "monthly_pass_valid": monthly_valid,
            "fee": round(amount, 2),
            "receipt_number": payment.receipt_number,
        }
        self.parking_records.append(record)

        return {
            "success": True,
            "message": f"Vehicle {plate} exited successfully.",
            "space_id": space_id,
            "duration_hours": round(duration, 2),
            "fee": round(amount, 2),
            "monthly_pass_valid": monthly_valid,
            "receipt": receipt,
            "ticket_id": ticket.ticket_id if ticket else None,
        }

    # ---- Reporting ----

    def get_system_report(self) -> Dict[str, Any]:
        occupied_count = len(self.occupied_spaces)
        available_count = len(self.available_spaces)
        active_monthly = sum(1 for p in self.monthly_pass_holders.values() if p.is_valid())
        total_revenue = 0.0

        # Sum fees from exit records (hourly only). Monthly is 0 per visit here.
        for rec in self.parking_records:
            if rec.get("event") == "exit":
                total_revenue += float(rec.get("fee", 0.0))

        return {
            "total_spaces": self.total_spaces,
            "occupied_spaces": occupied_count,
            "available_spaces": available_count,
            "active_vehicles": list(self.active_vehicles.keys()),
            "active_monthly_passes": active_monthly,
            "total_revenue_collected": round(total_revenue, 2),
            "records_count": len(self.parking_records),
        }



# Optional CLI demo (for testing)


def _print_menu() -> None:
    print("\nUrban City Parking - Menu")
    print("1) Vehicle Entry")
    print("2) Vehicle Exit")
    print("3) Add/Renew Monthly Pass")
    print("4) System Report")
    print("5) Exit")


def main() -> None:
    parking = ParkingSystem(total_spaces=300)

    while True:
        _print_menu()
        choice = input("Choose an option: ").strip()

        if choice == "1":
            plate = input("Plate number: ").strip()
            vtype = input("Vehicle type (Car/Motorcycle): ").strip()
            result = parking.vehicle_entry(plate, vtype)
            print(result["message"])
            if result.get("success"):
                print(f"Assigned space: {result['space_id']} (Zone {result['zone']})")
                print(f"Entry time: {result['entry_time']}")
                print(f"Ticket: {result['ticket']}")
                print(f"Monthly pass valid: {result['monthly_pass_valid']}")

        elif choice == "2":
            plate = input("Plate number: ").strip()
            result = parking.vehicle_exit(plate)
            print(result["message"])
            if result.get("success"):
                print(f"Duration (hours): {result['duration_hours']}")
                print(f"Fee: ${result['fee']:.2f}")
                print(f"Monthly pass valid: {result['monthly_pass_valid']}")
                print(result["receipt"])

        elif choice == "3":
            plate = input("Plate number for monthly pass: ").strip()
            result = parking.add_monthly_pass(plate)
            print(f"Monthly pass {result['status']} for {result['plate_number']}")
            print(f"Valid from: {result['valid_from']}")
            print(f"Valid to: {result['valid_to']}")
            print(result["receipt"])

        elif choice == "4":
            report = parking.get_system_report()
            print("\n--- System Report ---")
            for k, v in report.items():
                print(f"{k}: {v}")

        elif choice == "5":
            print("Goodbye!")
            break

        else:
            print("Invalid option. Please choose 1-5.")


if __name__ == "__main__":
    main()
