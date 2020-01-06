```
  double helpOffset(double a,double b){

    print("====${a}");
    if(a>=540&&b>0){
      return a+2;
    }else if((a>=360&&a<540) && b>0){
      return a+40;
    }else if((a<=360&&a>180) && b<0){
      return a-40;
    }else if(a<=180 &&b<0){
      return a-2;
    } else if((a<180&&a>=0) && b>0){
      return a+40;
    }else if(a>=180 && b>0){
      return a+2;
    }
  }
```
