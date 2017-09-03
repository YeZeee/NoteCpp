# Summary about mutex

## Type of mutex

|Type|Function|Attribute|
|-|-|-|
|std::mutex|For basic synchronization|exclusive<br>non-recursive<br>not copyable and moveable|
|std::shared_mutex|For rare update synchronization|exclusive or shared<br>non-recursive<br>not copyable and moveable|
|std::recursive_mutex|For recursive locking|exclusive<br>recursive<br>not copyable and moveable|
|std::timed_mutex|For timing try locking|exclusive<br>non-recursive<br>not copyable and moveable<br>timing|
|std::shared_timed_mutex|For rare update synchronization and timing|exclusive or shared<br>non-recursive<br>not copyable and moveable<br>timing|
|std::recursive_timed_mutex|For recursive locking and timing|exclusive<br>recursive<br>not copyable and moveable<br>timing|

## RAII style lock management template

|Type|Function|Attribute|
|-|-|-|
|std::lock_guard|For basic RAII style exclusive lock management||
|std::shared_lock|For shared lock||
|std::unique_lock|For movable mutex ownership wraper||
|std::scoped_lock|For For exclusive lock avoiding deadlock||