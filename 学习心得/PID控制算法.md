# PID算法

## 位置式PID

```c
// PID控制器结构体(位置式)  
typedef struct {  
    double kp;         // 比例系数  
    double ki;         // 积分系数  
    double kd;         // 微分系数  
    double prev_error; // 上一次的误差  
    double integral;   // 误差的积分  
    double target;     // 目标值  
} PID_Controller; 

// output = Kp * error + ki * error累加和 + kd * (当前error与上一个error的差)	(位置式PID)
int32_t adc_pidUpdate(PID_Controller *pid, int32_t nowV)
{
	// 比例
	double error = pid->target - nowV;
	double p = pid->kp * error;
	// 积分
	double integral = pid->integral;
	integral += error;
	// 防止积分过饱和
	if (integral > ADC_MULTI(3) || integral < ADC_MULTI(-3)) {
		if (integral > 0) integral = ADC_MULTI(3);
		else integral = ADC_MULTI(-3);
	}
	pid->integral = integral;
	double i = pid->ki * pid->integral;
	// 微分
	double derivative = error - pid->prev_error;
	double d = pid->kd * derivative;

	pid->prev_error = error;

	double output = p + i + d;

	return (int32_t)output;
}
```



## 增量式PID

```c
// 增量式PID
// PID控制器结构体 
typedef struct {  
    double kp;         // 比例系数  
    double ki;         // 积分系数  
    double kd;         // 微分系数  
    double prev_error; // 上上一次的误差  
    double last_error; // 上一次的误差
    double error;	   // 误差
    double target;     // 目标值  
    double uk;		   // 上次控制量
} PID_Add_Controller; 

int32_t adc_pidAddUpdate(PID_Add_Controller *pid, int32_t nowV)
{
	double error = pid->target - nowV;
	double errorChange = error - pid->last_error;
	double p = pid->kp * errorChange;

	double i = pid->ki * error;
	double derivative = error - 2 * pid->last_error + pid->prev_error;
	double d = pid->kd * derivative;

	pid->prev_error = pid->last_error;
	pid->last_error = pid->error;
	pid->error = error;

	double output = p + i + d;
	/* output += pid->uk;

	output = HX_MAX(output, -ADC_MULTI(5));
	output = HX_MIN(output, ADC_MULTI(5));
	
	pid->uk = output; */
	return (int32_t) output;
}
```

