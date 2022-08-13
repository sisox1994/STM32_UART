# STM32 UART  



## "HAL_UART_Transmit()"  VS  "HAL_UART_Transmit_IT ()"

HAL_UART_Transmit()  VS  HAL_UART_Transmit_IT () 之間究竟有什麼差別?

HAL_UART_Transmit  會占用 CPU 的時序去處理 UART TX 傳輸的動作，直到傳送完畢才會繼續跑原本的程式。

HAL_UART_Transmit_IT 呼叫了之後，會在CPU有空的時候行分好幾次丟完，時序設定夠快的話，感覺就像是在背景執行，不影響原本的工作。



```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout)
```



```c
HAL_StatusTypeDef HAL_UART_Transmit_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
```



# HAL_UART_Transmit()

兩者參數差別只有 Timeout，而Timeout是做什麼用的呢?

A:  HAL_UART_Transmit 是 blocking function，為了防止TX資料傳輸時間太久導致主迴圈的工作都塞住了，因此設置Timeout限制TX資料必須在時間內傳完，如果沒傳完就會直接強制回到主迴圈繼續執行，TX資料可能就會被截掉。



```c
void main(){
    
    uint8_t rawData[] = "Limit:    ,Target:    ,nBUS_mA:     ";
    
    while(1){
        
    	if(T_1s_Flag){
     	    T_1s_Flag = 0;
            
            RS485_TX_Transform( rawData, sizeof(rawData) );
    	}
        
        if(T_1ms_Flag){      
            T_1ms_Flag = 0; 
            
            HAL_GPIO_TogglePin(GPIOB,  GPIO_PIN_0);
        }
    }
}

void RS485_TX_Transform( uint8_t* txbuf, uint8_t size ){
    
    DEControl(1);    
    HAL_UART_Transmit( &RS485_huart, txbuf , size, 50 ); // timeout 50ms
    DEControl(0);
    
    __HAL_UART_ENABLE_IT(&RS485_huart, UART_IT_RXNE);  
   
}
```



<img src="/1.png" alt="1"  />





# HAL_UART_Transmit_IT()



HAL_UART_Transmit_IT 雖然可以讓MCU達成類似背景執行UART TX傳輸的功能，但使用上有一些小細節需要注意!!

例如:  HAL_UART_Transmit_IT 字面上意思就是把 UART TX 的工作丟給系統中斷去做，因此呼叫 HAL_UART_Transmit_IT時，並不是馬上把我們要傳輸的TX Raw Data一次丟完，而只是先告知系統我們要傳輸的資料在哪個"記憶體位址"，CPU會在有空閒的時候自動切換到中斷程式把TX 資料丟完，因為切換速度很快，感覺就像是在背景同步執行一樣，因此 HAL_UART_Transmit_IT 內的 uint8_t *pData參數，千萬不能使用STACK記憶體位置，不然實際傳出來的rawData一定會跟你想的不太一樣，切記使用前先將rawData複製到一個專門給TX Data使用的 GLOBAL buffer裡面，這樣資料才會正確!!!!



另外如果要實作 RS485的CTRL Pin傳輸完畢拉LOW的功能，可以寫一個小function在main迴圈抓取ISR Register的UART_FLAG_TC判斷是否傳輸完畢。

<img src="/3.png" alt="3"  />



```c
void DEControl_Task();

void main(){   
    
    uint8_t rawData[] = "Limit:    ,Target:    ,nBUS_mA:     ";
    
    while(1){
        
    	if(T_1s_Flag){
     	    T_1s_Flag = 0;
            
            RS485_TX_Transform( rawData, sizeof(rawData) );
    	}
        
        if(T_1ms_Flag){      
            T_1ms_Flag = 0; 
            
            HAL_GPIO_TogglePin(GPIOB,  GPIO_PIN_0);
        }
        
        DEControl_Task();
    }
}

uint8_t global_txbuf[60];

void RS485_TX_Transform( uint8_t* txbuf, uint8_t size ){
    
    DEControl(1);    
    
    memcpy( global_txbuf , txbuf , size);
    HAL_UART_Transmit_IT(&RS485_huart, global_txbuf , size);
    
    __HAL_UART_ENABLE_IT(&RS485_huart, UART_IT_RXNE);  
   
}

void DEControl_Task(){
  
    // TC: Transmission complete
    // 0: Transmission is not complete
    // 1: Transmission is complete  
    uint32_t isrflags = READ_REG(RS485_huart.Instance->ISR);
  
    //---- Transmit OK 485 Contrl Pin need to reset   ---
    if ((isrflags & UART_FLAG_TC) != 0U){
        DEControl(0);
    }
}
```

<img src="/2.png" alt="2"  />
