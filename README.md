# bot_mql5_trading_rsi
bot de trading hecho en mql5 , con funciones de rsi toma entrada de compra y venta al cumplirse las condiciones configuradas por el usuario.
#property copyright "Copyright 2024, MetaQuotes Ltd."
#property link      "https://www.mql5.com"
#property version   "1.00"
//+------------------------------------------------------------------+
//| include                                                          |
//+------------------------------------------------------------------+
#include  <Trade\Trade.mqh>
//+------------------------------------------------------------------+
//| Inputs                                                           |
//+------------------------------------------------------------------+
static input long    InpMagicNumber = 546812;    // numero magico
static input double  InpLotSize     = 0.01;      // lotaje
input int            InpRsiPeriod   = 21;        // periodo rsi
input int            InpRsiLevel    = 70;        // rsi nivel superior
input int            InpMAperiod    = 21;        // ma periodo
input ENUM_TIMEFRAMES InpMATimeframe = PERIOD_H1; // MA periodo de tiempo
input int             InpStopLoss   = 200;       // stop loss 0=off
input int            InpTakeProfit  = 100;       // take profit 0=off
input bool           InpCloseSignal = false;     // cerrar posicion contraria

//+------------------------------------------------------------------+
//| variables globales                                               |
//+------------------------------------------------------------------+
int handleRSI;
int handleMA;
double bufferRSI[];
double bufferMA[];
MqlTick currentTick;
CTrade trade;

int OnInit()
  {
   
   if(InpMagicNumber<=0){
   Alert("número mágico ingresado incorrectamente <=0");
   return INIT_PARAMETERS_INCORRECT;
   }
   
   if(InpLotSize<=0 || InpLotSize>10){
   Alert("tamaño de lote <=0 o >10");
   return INIT_PARAMETERS_INCORRECT;
   }
   
   if(InpRsiPeriod<=1){
   Alert("stop loss incorecto <=1");
   return INIT_PARAMETERS_INCORRECT;
   }
   
    if(InpRsiLevel>=100 || InpRsiLevel<=50){
   Alert("tamaño de lote >=100 o <=50");
   return INIT_PARAMETERS_INCORRECT;
   }
   if(InpMAperiod<1){
   Alert("MA periodo <1");
   return INIT_PARAMETERS_INCORRECT;
   }
   
   if(InpStopLoss<0){
   Alert("stop loss  <0");
   return INIT_PARAMETERS_INCORRECT;
   }
   if(InpTakeProfit<0){
   Alert("stop loss  <0");
   return INIT_PARAMETERS_INCORRECT;
   }
   // colocar numero magico
   trade.SetExpertMagicNumber(InpMagicNumber);
   
   //crear indicador handle
   handleRSI = iRSI(_Symbol,PERIOD_CURRENT,InpRsiPeriod,PRICE_OPEN);
   if(handleRSI== INVALID_HANDLE){
      Alert("falla al crear el indicador handleRSI");
      return INIT_FAILED;
   }
   //crear indicador handle
   handleMA = iMA(_Symbol,InpMATimeframe,InpMAperiod,0,MODE_SMA,PRICE_OPEN);
   if(handleMA== INVALID_HANDLE){
      Alert("falla al crear el indicador handleMA");
      return INIT_FAILED;
   }
   
   // colocar serie de buffer
   ArraySetAsSeries(bufferRSI,true);
   ArraySetAsSeries(bufferMA,true);
   
   return(INIT_SUCCEEDED);
  }
//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason) 
  {
   
   // liberar indicador handles
   if(handleRSI!=INVALID_HANDLE) {IndicatorRelease(handleRSI);}
   if(handleMA!=INVALID_HANDLE) {IndicatorRelease(handleMA);}
   
  }
//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
  {
   
   //comprobar si el tick actual es un tick abierto de barra nueva
   
   if(!IsNewBar()){return;}
   
   //obtener tickeck actual
   if(!SymbolInfoTick(_Symbol,currentTick)){Print("failed to get current tick"); return;}
   
   // obtener valores del rsi
   int values = CopyBuffer(handleRSI,0,0,2,bufferRSI);
   if(values!=2){
      Print("fallo en obtener valores del indicado");
      return;
   }
   // obtener valores del MA
   values = CopyBuffer(handleMA,0,0,1,bufferMA);
   if(values!=1){
      Print("fallo en obtener valores del indicador MA");
      return;
   }
   
   
   Comment("bufferRSI[0]:",bufferRSI[0],
           "\nbufferRSI[1]:",bufferRSI[1],
            "\nbufferMA[0]:",bufferMA[0]);
   
   //conteo de operaciones abiertas
   int cntBuy,cntSell;
   if(!CountOpenPositions(cntBuy,cntSell)){return;}
   
   // verifia posiiciones de compra
   if(cntBuy==0 && bufferRSI[1]>=(100-InpRsiLevel) && bufferRSI[0]<(100-InpRsiLevel) && currentTick.ask>bufferMA[0]){
   
      
      if(InpCloseSignal){if(!ClosePositions(2)){return;}}
      double sl = InpStopLoss==0 ? 0 : currentTick.bid -InpStopLoss * _Point;
      double tp = InpTakeProfit==0 ? 0 : currentTick.bid +InpTakeProfit * _Point;
      if(!NormalizePrice(sl)){return;}
      if(!NormalizePrice(tp)){return;}
      
      trade.PositionOpen(_Symbol,ORDER_TYPE_BUY,InpLotSize,currentTick.ask,sl,tp,"RSI MA FILTRAR EA");
   }
   
   // verifia posiiciones de venta
   if(cntSell==0 && bufferRSI[1]<=InpRsiLevel && bufferRSI[0]>InpRsiLevel && currentTick.bid<bufferMA[0]){
   
      if(InpCloseSignal){if(!ClosePositions(1)){return;}}
      double sl = InpStopLoss==0 ? 0 : currentTick.ask +InpStopLoss * _Point;
      double tp = InpTakeProfit==0 ? 0 : currentTick.ask -InpTakeProfit * _Point;
      if(!NormalizePrice(sl)){return;}
      if(!NormalizePrice(tp)){return;}
      
      trade.PositionOpen(_Symbol,ORDER_TYPE_SELL,InpLotSize,currentTick.bid,sl,tp,"RSI MA FILTRAR EA");
   }
}

//+------------------------------------------------------------------+
//| funciones personalizadas                                         |
//+------------------------------------------------------------------+

//Comprueba si tenemos una barra abierta.
bool IsNewBar(){
   static datetime previusTime = 0;
   datetime currentTime = iTime(_Symbol,PERIOD_CURRENT,0);
   if(previusTime!=currentTime){
   previusTime=currentTime;
    return true;
   }
   return false;
}  
// cuenta de posiciones cerradas
bool ClosePositions (int all_buy_sell){

   int total = PositionsTotal();
   for(int i= total-1;i>=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("nose pudo obtener el ticket de la posicion"); return false;}
      if(!PositionSelectByTicket(ticket)){Print("fallo en la seleccion de posicion");return false;}
      long magic;
      if(!PositionGetInteger(POSITION_MAGIC,magic)){Print("fallo en obtener las posicion de numero magico"); return false;}
      if(magic==InpMagicNumber){
         long type;
         if(!PositionGetInteger(POSITION_TYPE,type)){Print("no se pudo obtener tipo de posicion");return false;}
         if(all_buy_sell==1 && type==POSITION_TYPE_SELL){continue;}
         if(all_buy_sell==2 && type==POSITION_TYPE_BUY){continue;}
         trade.PositionClose(ticket);
         if(trade.ResultRetcode()!=TRADE_RETCODE_DONE){
            Print("fallo en cerrar la posicion:",(string)ticket,
            "resultado:",(string)trade.ResultRetcode(),"i",trade.CheckResultRetcodeDescription());
         }
       
      
      }
   }

   return true;
}


// normalizar precio 
bool NormalizePrice (double &price){
   
   double tickSize=0;
   if(!SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE,tickSize)){
      Print("no se pudo obtener el tamaño del boleto");
      return false;
   }
   price = NormalizeDouble(MathRound(price/tickSize)*tickSize,_Digits);

   return true;
}

// cuenta de posiciones abiertas
bool CountOpenPositions (int &cntBuy, int &cntSell){

   cntBuy = 0;
   cntSell = 0;
   int total = PositionsTotal();
   for(int i= total-1;i>=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("nose pudo obtener el ticket de la posicion"); return false;}
      if(!PositionSelectByTicket(ticket)){Print("fallo en la seleccion de posicion");return false;}
      long magic;
      if(!PositionGetInteger(POSITION_MAGIC,magic)){Print("fallo en obtener las posicion de numero magico"); return false;}
      if(magic==InpMagicNumber){
         long type;
         if(!PositionGetInteger(POSITION_TYPE,type)){Print("no se pudo obtener tipo de posicion");return false;}
         if(type==POSITION_TYPE_BUY){cntBuy++;}
         if(type==POSITION_TYPE_SELL){cntSell++;}
      
      }
   }

   return true;
}
