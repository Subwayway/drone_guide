---
layout: post
title: MAVLINK 메세지
category: PX4
---
- test_task에서 시스템 타임을 계속해서 test_topic에 publish함.
- test_topic에서 업데이트된 값을 subscribe하여 MAVLINK DEBUG msg를 통해 QGC로 전송함.
- QGC의 MAVLINK Inspector를 통해 전송된 값을 확인함.


uORB를 통해 publish/subscribe를 할 topic을 msg/ 디렉토리에 추가.
```c

uint64 timestamp		# time since system start (microseconds)

uint64 timestamp_sample		# the timestamp of the raw data (microseconds)

float32[3] test_data			# test_topic test data
```

추가한 topic을 msg/CMakeLists.txt에 추가.
```c
set(msg_files
    test_topic.msg
)
```

topic에 publish할 태스크를 src/example/px4_test_task에 추가
px4_test_task.cpp
```cpp
#include "px4_test_task.hpp"

px4_test_task::px4_test_task() :
	ModuleParams(nullptr),
	ScheduledWorkItem(MODULE_NAME, px4::wq_configurations::test1)
{
}

px4_test_task::~px4_test_task()
{
	perf_free(_loop_perf);
	perf_free(_loop_interval_perf);
}

bool px4_test_task::init()
{
	// alternatively, Run on fixed interval
	ScheduleOnInterval(5000_us); // 2000 us interval, 200 Hz rate

	return true;
}

void px4_test_task::Run()
{
	if (should_exit()) {
		ScheduleClear();
		exit_and_cleanup();
		return;
	}

	perf_begin(_loop_perf);
	perf_count(_loop_interval_perf);

	// Check if parameters have changed
	if (_parameter_update_sub.updated()) {
		// clear update
		parameter_update_s param_update;
		_parameter_update_sub.copy(&param_update);
		updateParams(); // update module parameters (in DEFINE_PARAMETERS)
	}


	// Example
	//  update test_topic to check arming state
	struct test_topic_s test_topic;
	if (_orb_test_sub.updated()) {

		if (_orb_test_sub.copy(&test_topic)) {

			//do something...
		}
	}


	// Example
	//  publish some data
	test_topic_s data;
	data.test_data[0] = hrt_absolute_time()+1;
	data.test_data[1] = hrt_absolute_time()+2;
	data.test_data[2] = hrt_absolute_time()+3;
	data.timestamp = hrt_absolute_time();
	_orb_test_pub.publish(data);


	perf_end(_loop_perf);
}

int px4_test_task::task_spawn(int argc, char *argv[])
{
	px4_test_task *instance = new px4_test_task();

	if (instance) {
		_object.store(instance);
		_task_id = task_id_is_work_queue;

		if (instance->init()) {
			return PX4_OK;
		}

	} else {
		PX4_ERR("alloc failed");
	}

	delete instance;
	_object.store(nullptr);
	_task_id = -1;

	return PX4_ERROR;
}

int px4_test_task::print_status()
{
	perf_print_counter(_loop_perf);
	perf_print_counter(_loop_interval_perf);
	return 0;
}

int px4_test_task::custom_command(int argc, char *argv[])
{
	return print_usage("unknown command");
}

int px4_test_task::print_usage(const char *reason)
{
	if (reason) {
		PX4_WARN("%s\n", reason);
	}

	PRINT_MODULE_DESCRIPTION(
		R"DESCR_STR(
### Description
Example of a simple module running out of a work queue.

)DESCR_STR");

	PRINT_MODULE_USAGE_NAME("px4_test_task", "driver");
	PRINT_MODULE_USAGE_COMMAND("start");
	PRINT_MODULE_USAGE_DEFAULT_COMMANDS();

	return 0;
}

extern "C" __EXPORT int px4_test_task_example_main(int argc, char *argv[])
{
	return px4_test_task::main(argc, argv);
}
```

px4_test_task.hpp
```cpp
#pragma once

#include <px4_platform_common/defines.h>
#include <px4_platform_common/module.h>
#include <px4_platform_common/module_params.h>
#include <px4_platform_common/posix.h>
#include <px4_platform_common/px4_work_queue/ScheduledWorkItem.hpp>

#include <drivers/drv_hrt.h>
#include <lib/perf/perf_counter.h>

#include <uORB/Publication.hpp>
#include <uORB/Subscription.hpp>
#include <uORB/SubscriptionCallback.hpp>
#include <uORB/topics/test_topic.h>
#include <uORB/topics/parameter_update.h>

using namespace time_literals;

class px4_test_task : public ModuleBase<px4_test_task>, public ModuleParams, public px4::ScheduledWorkItem
{
public:
	px4_test_task();
	~px4_test_task() override;

	/** @see ModuleBase */
	static int task_spawn(int argc, char *argv[]);

	/** @see ModuleBase */
	static int custom_command(int argc, char *argv[]);

	/** @see ModuleBase */
	static int print_usage(const char *reason = nullptr);

	bool init();

	int print_status() override;

private:
	void Run() override;

	// Publications
	uORB::Publication<test_topic_s> _orb_test_pub{ORB_ID(test_topic)};

	// Subscriptions
	uORB::SubscriptionInterval         _parameter_update_sub{ORB_ID(parameter_update), 1_s}; // subscription limited to 1 Hz updates
	uORB::Subscription                 _orb_test_sub{ORB_ID(test_topic)};          // regular subscription for additional data

	// Performance (perf) counters
	perf_counter_t	_loop_perf{perf_alloc(PC_ELAPSED, MODULE_NAME": cycle")};
	perf_counter_t	_loop_interval_perf{perf_alloc(PC_INTERVAL, MODULE_NAME": interval")};

	// Parameters
	DEFINE_PARAMETERS(
		(ParamInt<px4::params::SYS_AUTOSTART>) _param_sys_autostart,   /**< example parameter */
		(ParamInt<px4::params::SYS_AUTOCONFIG>) _param_sys_autoconfig  /**< another parameter */
	)


	bool _armed{false};
};
```

CMakeLists.txt
```txt
px4_add_module(
	MODULE examples__px4_test_task
	MAIN px4_test_task_example
	COMPILE_FLAGS
		#-DDEBUG_BUILD   # uncomment for PX4_DEBUG output
		#-O0             # uncomment when debugging
	SRCS
		px4_test_task.cpp
		px4_test_task.hpp
	DEPENDS
		px4_work_queue
	)

```

빌드에 test_task를 추가. boards/px4/fmu-v3/default.cmake
```c
EXAMPLES
		fixedwing_control # Tutorial code from https://px4.io/dev/example_fixedwing_control
		hello
		hwtest # Hardware test
		#matlab_csv_serial
		px4_mavlink_debug # Tutorial code from http://dev.px4.io/en/debug/debug_values.html
		px4_simple_app # Tutorial code from http://dev.px4.io/en/apps/hello_sky.html
		rover_steering_control # Rover example app
		uuv_example_app
		work_item
		px4_test_task
```

px4가 구동을 시작할때 자동으로 태스크가 실행되도록 스타트 스크립트에 test_task를 추가.
ROMFS/px4fmu_common/init.d/rcS
```bash
px4_test_task_example start
```

MAVLINK debug msg를 이용하기 위해 기존 debug msg 함수에 test_topic의 값을 subscribe하여 전송하게끔함.
src/modules/mavlink/mavlink_messages.cpp (1.11.2 버전기준)
최신버전의 경우 MAVLINK msg들의 함수 정의는 src/modules/mavlink/streams 폴더에 정의되어있음.
```cpp
#include <uORB/topics/test_topic.h>

class MavlinkStreamDebug : public MavlinkStream
{
public:
	const char *get_name() const override
	{
		return MavlinkStreamDebug::get_name_static();
	}

	static constexpr const char *get_name_static()
	{
		return "DEBUG";
	}

	static constexpr uint16_t get_id_static()
	{
		return MAVLINK_MSG_ID_DEBUG;
	}

	uint16_t get_id() override
	{
		return get_id_static();
	}

	static MavlinkStream *new_instance(Mavlink *mavlink)
	{
		return new MavlinkStreamDebug(mavlink);
	}

	unsigned get_size() override
	{
		return _debug_sub.advertised() ? MAVLINK_MSG_ID_DEBUG_LEN + MAVLINK_NUM_NON_PAYLOAD_BYTES : 0;
	}

private:
	uORB::Subscription _debug_sub{ORB_ID(test_topic)};

	/* do not allow top copying this class */
	MavlinkStreamDebug(MavlinkStreamDebug &) = delete;
	MavlinkStreamDebug &operator = (const MavlinkStreamDebug &) = delete;

protected:
	explicit MavlinkStreamDebug(Mavlink *mavlink) : MavlinkStream(mavlink)
	{}

	bool send(const hrt_abstime t) override
	{
		test_topic_s debug;

		if (_debug_sub.update(&debug)) {
			mavlink_debug_t msg{};
			msg.time_boot_ms = debug.timestamp / 1000ULL;
			msg.ind = debug.test_data[0];
			msg.value = debug.test_data[1];

			mavlink_msg_debug_send_struct(_mavlink->get_channel(), &msg);

			return true;
		}

		return false;
	}
};
```

uorb top 명령을 통해 topic이 활성화 되어있는지 확인.
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/116046810-642b1f00-a6ae-11eb-91ae-98a573b767c4.png" width="80%"></p>

listener test_topic 명령을 통해 topic에 값이 업데이트 되는지 확인.
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/116046881-7442fe80-a6ae-11eb-92b8-1f3740b963c4.png" width="80%"></p>

MAVLINK INSPECTOR를 통해 MAVLINK debug msg가 test_topic의 값을 GCS로 전송되는지 확인.
<p align="center"><img src="https://user-images.githubusercontent.com/48232366/116046933-82911a80-a6ae-11eb-974b-b9ab988e6243.png" width="80%"></p>