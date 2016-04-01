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
