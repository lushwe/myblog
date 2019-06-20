## 本文简单介绍一下 `FactoryBean`

- `FactoryBean` 是 `Spring` `Bean` 包中一个接口（`org.springframework.beans.factory.FactoryBean`）

- `FactoryBean` 接口有三个方法
  - `getObject()`
  - `getObjectType()`
  - `isSingleton()`

- 如果一个对象实现了 `FactoryBean` 接口，那么该对象将不会直接作为一个 `Bean` 实例暴露，而是一个创建该对象的工厂，什么意思呢？接下来用代码解释
  - i 假设如下代码， `CarFactoryBean` 实现 `FactoryBean`
  ```java
  @Component("car")
  public class CarFactoryBean implements FactoryBean<Car> {
  
      private String carInfo = "超级跑车,400,2000000";
  
      @Override
      public Car getObject() {
  
          Car car = new Car();
          String[] infos = carInfo.split(",");
          car.setBrand(infos[0]);
          car.setMaxSpeed(Integer.parseInt(infos[1]));
          car.setPrice(Double.parseDouble(infos[2]));
  
          return car;
      }
  
      @Override
      public Class<Car> getObjectType() {
          return Car.class;
      }
  
      @Override
      public boolean isSingleton() {
          return true;
      }
  
      public String getCarInfo() {
          return carInfo;
      }
  
      public void setCarInfo(String carInfo) {
          this.carInfo = carInfo;
      }
  }
  ```
  - ii 进行测试
  ```java
  @SpringBootApplication
  public class Application {
  
      public static void main(String[] args) {
  
          ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
  
          try {
              Object car = context.getBean("car");
              System.out.println("car = " + car);
          } catch (Exception e) {
              e.printStackTrace();
          }
  
          context.close();
      }
  }
  ```
  - iii 日志打印如下
  ```
  car = com.lushwe.controller.Car@6b85300e
  ```
  由此看来 `context.getBean("car")` 获取的是 `Car` 对象实例，并非 `CarFactoryBean` 对象实例；那如何获取 `CarFactoryBean` 对象实例呢，有两种方法，如下

  ```java
  try {
      Object car1 = context.getBean("&car"); // 方法一： bean 名称获取
      Object car2 = context.getBean(CarFactoryBean.class); // 方法二：bean 类型获取
      System.out.println("car1 = " + car1);
      System.out.println("car2 = " + car2);
  } catch (Exception e) {
      e.printStackTrace();
  }
  ```
  日志打印如下

  ```
  car1 = com.lushwe.controller.CarFactoryBean@79ab3a71
  car2 = com.lushwe.controller.CarFactoryBean@79ab3a71
  ```

- 总结

  `CarFactoryBean` 实现 `FactoryBean` ，单纯根据  `CarFactoryBean` 对应的名称获取，获取到是其方法 `getObject()` 返回的对象实例，并非 `CarFactoryBean` 对象实例，要获取 `CarFactoryBean` 对象实例：

  - 第一，根据名称获取，要加前缀 `&` ，这里使用 `&car`
  - 第二，根据类型获取
  
  
