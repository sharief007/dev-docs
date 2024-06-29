
Design a meeting scheduler.
- There are n meeting rooms available each with a different capacity.
- Book any available meeting room for the given interval. (start, end, attendees).
- Send notifications to the users invited to the meeting.

```python
import bisect
from abc import abstractmethod, ABC
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from random import randint
from typing import List, Dict, Tuple, Set

@dataclass
class User:
    id: int
    name: str
    
    @classmethod
    def notify(cls, meeting_request):
        meeting_request.acknowledge(MeetingResponse(randint(1, 2)))
        

class MeetingResponse(Enum):
    ACCEPTED = 1
    REJECTED = 2


@dataclass
class Meeting:
    start_time: datetime
    end_time: datetime
    participants: List[User]
    attendees: Dict[User, MeetingResponse] = field(default_factory=dict)
    subject: str = ''
    description: str = ''


@dataclass
class MeetingRequest:
    recipient: User
    meeting: Meeting

    def acknowledge(self, response: MeetingResponse):
        self.meeting.attendees[self.recipient] = response


@dataclass
class MeetingRoom:
    @dataclass
    class Event:
        time_stamp: datetime
        opening: bool

    id: int
    capacity: int
    _meetings: Set[Meeting] = field(default_factory=set)
    _time_line: List['MeetingRoom.Event'] = field(default_factory=list)

    def is_available(self, start_time: datetime, end_time: datetime) -> bool:
        for (slot_start, slot_end) in self.free_slots:
            if slot_start <= start_time and slot_end >= end_time:
                return True
        return False

    @property
    def free_slots(self) -> List[Tuple[datetime, datetime]]:
        meetings = 0
        prev_close = None
        result = []
        for event in self._time_line:
            if event.opening:
                if meetings == 0 and prev_close and prev_close < event.time_stamp:
                    result.append((prev_close, event.time_stamp))
                meetings += 1
            else:
                prev_close = event.time_stamp
                meetings -= 1
        return result

    def can_accommodate(self, count: int) -> bool:
        return count <= self.capacity

    def add_booking(self, meeting: Meeting) -> bool:
        if not self.is_available(meeting.start_time, meeting.end_time):
            return False
        if not self.can_accommodate(len(meeting.participants)):
            return False

        bisect.insort(self._time_line, self.Event(meeting.start_time, True))
        bisect.insort(self._time_line, self.Event(meeting.end_time, False))
        self._meetings.add(meeting)
        return True


class SchedulerObserver(ABC):
    @abstractmethod
    def update(self, meeting_request: MeetingRequest):
        pass


@dataclass
class MeetingScheduler:
    _rooms: List[MeetingRoom] = field(default_factory=list)
    _users: List[User] = field(default_factory=list)

    def add_user(self, user: User):
        self._users.append(user)

    def add_room(self, room: MeetingRoom):
        self._rooms.append(room)

    def schedule(self, meeting: Meeting) -> bool:
        for room in self._rooms:
            if room.add_booking(meeting):
                self._notify_participants(meeting)
                return True
        return False

    def _notify_participants(self, meeting: Meeting):
        for participant in meeting.participants:
            meeting_request = MeetingRequest(recipient=participant, meeting=meeting)
            participant.notify(meeting_request)


# Client code
if __name__ == "__main__":
    from datetime import timedelta

    scheduler = MeetingScheduler()

    # Adding users
    user1 = User(id=1, name="Alice")
    user2 = User(id=2, name="Bob")
    scheduler.add_user(user1)
    scheduler.add_user(user2)

    # Adding meeting rooms
    room1 = MeetingRoom(id=1, capacity=10)
    room2 = MeetingRoom(id=2, capacity=20)
    scheduler.add_room(room1)
    scheduler.add_room(room2)

    # Scheduling a meeting
    meeting = Meeting(
        start_time=datetime.now(),
        end_time=datetime.now() + timedelta(hours=1),
        participants=[user1, user2],
        subject="Team Meeting",
        description="Discuss project updates"
    )

    if scheduler.schedule(meeting):
        print("Meeting scheduled successfully")
    else:
        print("Failed to schedule the meeting")
```