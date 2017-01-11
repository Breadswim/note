# SpringMVC Controller转发

## 使用RedirectAttributes携带参数

```java
@RequestMapping("/from")
public ModelAndView fromController(TestVo vo, RedirectAttributes ra) {
    ModelAndView view = new ModelAndView();
    ra.addAttribute("attr1", "valure1");
    view.setViewName("redirect:/dest");//这里不能用forward
  	
  	view.addObject("fromKey", "fromValue");
    return view;
}
```

转发时会把`RedirectAttributes`中的参数拼接到url上，如open?attr1=valure1

```java
@RequestMapping("/dest")
public ModelAndView destController(String id, String attr1) {
    ModelAndView view = new ModelAndView("optima/plan/planInfo");
    if (attr1 != null) {
        System.out.println(attr1);//output: valure1
    }
    view.addObject("toKey", "toValue");
    return view;
}
```



## 在前台页面可同时访问到两个ModelAndView中的属性。