##使用NSPredicate从对象数组中获得特定ID对象的方法：

* 创建NSPredicate, 格式`@"xxxID == %@", @(xxxID)`
* 使用NSArray的`filteredArrayUsingPredicate:`方法

```objc
- (AGCPointsMaxProgram *)programWithProgramID:(NSInteger)programID {
    NSPredicate *predicate = [NSPredicate predicateWithFormat:@"pointsMaxID == %@", @(programID)];
    NSArray<AGCPointsMaxProgram *> *programs = [self.mockPrograms filteredArrayUsingPredicate:predicate];
    if (programs.count == 1) {
        return programs.firstObject;
    }
    return nil;
}
```
